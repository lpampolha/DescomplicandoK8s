# DescomplicandoK8s
## Baixando a imagem do Debian

sudo debootstrap stable /debian http://deb.debian.org/debian

## Montando o container

unshare --mount --pid --net --uts --ipc --map-root-user --user --fork chroot /debian /bin/bash

## Montando diretórios 

mount -t proc none /proc/
ps -ef
ip a
mount -t sysfs none /sys/
mount -t tmpfs none /tmp/

## CGROUP

### Instalando pacotes
sudo apt-get install cgroup-tools

cgcreate -g cpu,memory,blkio,devices,freezer:giropops
ls /sys/fs/cgroup/cpu/giropops

#### chamando novamente o container com o unshare, o PID do bash  fará referência ao do unshare, como sendo processo filho, no HOST.

#### Aqui a gente associa os controles do container ao HOST, para melhor gerenciar o processo no cgroup que acabamos de montar
cgclassify -g cpu,memory,blkio,devices,freezer:giropops "PID bash"

#### Para conferir
cat /sys/fs/cgroup/cpu/giropops/cgroup.procs

#### Verificando o número de cpus
getconf _NPROCESSORS_ONLN (mesma informação dada com cat /proc/cpuinfo)

#### Setando o limite de cpu do container
cgset -r cpu.cfs_quota_us=1000 giropops

#### verificando se o setar de 1% funcionou
cat /sys/fs/cgroup/cpu/giropops/cpu.cfs_quota_us

#### Testando
#### sem o stress, usaremos o dd
dd if=/dev/zero of=catota.img bs=8k count=256k

## Kubernetes
### Day-3
#### Formas de pegar a saída e criar um deployment
k get deployments.apps nginx-deployment -o yaml > teste.yaml

k create deployment --image nginx --replicas 3 nginx-deployment --dry-run=client -o yaml > deployment.yaml

#### Estratégias de atualizações nos Deployments (RollingUpdate e Recreate)
##### Para verificar status de modificações
k rollout status deployment -n giropops nginx-deployment

##### Para desfazer modificações
k rollout undo deployment -n giropops nginx-deployment

### Day-8
#### Para ver a saída de uma palavra como base64
echo -n "giropops" | base64 

#### Criando um yaml de secret opaque
  apiVersion: v1
  kind: Secret
  metadata:
    name: giropops-secret
  type: Opaque
  data: # Inicio dos dados
      username: SmVmZXJzb25fbGluZG8=
      password: Z2lyb3BvcHM=

#### Criando um secret opaque através da linha de comando
kubectl create secret generic giropops-secret --from-literal=username=SmVmZXJzb25fbGluZG8= --from-literal=password=Z2lyb3BvcHM=

#### Criando um secret para autenticar do Docker Hub, para acesso a imagens privadas 
Primeiro passo é codificar o conteúdo do arquivo ~/.docker/config.json, com o sequinte comando:
#base64 ~/.docker/config.json

Agora é criar o arquivo dockerhub-secret.yaml, com o seguinte conteúdo:

  kind: Secret
  apiVersion: v1
  metadata:
    name: docker-hub-secret
  type: kubernetes.io/dockerconsigjson
  data:
    .dockerconfigjson: | #aqui entra o valor do config.json codificado

Para baixar a imagem privada, será necessário usar o campo spec.imagePullSecrets

  kind: Pod
  apiVersion: v1
  metadata:
    name: nginx-secret
  spec:
    containers:
    - name: nginx-secret
      image: lpampolha/linuxtips-giropops-senhas:5.0
    imagePullSecrets:
    - name: docker-hub-secret

#### Criando secret TLS para armazenar certificado e chave TLS

openssl req -x509 -nodes -days 365 -newkey rsa:4096 -keyout chave-privada.key -out certificado.crt

- Agora é criar o secret

#kubectl create secret tls meu-servico-web-tls-secret --cert=certificado.crt --key=chave-privada.key

- Vamos utilizar o secret para ter nginx com https, usando o campo spec.tls

  kind: Pod
  apiVersion: v1
  metadata:
    name: nginx
    labels:
      app: nginx
  spec:
    containers:
      - name: nginx
        image: nginx
        ports:
          - containerPort: 80
          - containerPort: 443
        volumeMounts:
          - name: nginx-config-volume
            mountPath: /etc/nginx/nginx.conf
            subPath: nginx.conf
          - name: nginx-tls
            mountPath: /etc/nginx/tls
      volumes:
      - name: nginx-config-volume
        configMap:
          name: nginx-config
      - name: nginx-tls
        secret:
          secretName: meu-servico-web-tls-secret
          items:
            - key: certificado.crt
              path: certificado.crt
            - key: chave-privada.key
              path: chave-privada.key

#### Criando o ConfigMap

Criar arquivo nginx.conf
Depois é rodar o create para criar o configmap

  #kubectl create configmap nginx-config --from-file=nginx.conf

Também é possível criar o configmap através de um manifesto (nginx-configmap.yaml)

Agora é expor o pod
  #kubectl expose pod nginx

E dar um port-forward para testar
  #kubectl port-forward service/nginx 4443:443

### Day-9
#### Instalando o eksctl, que é uma ferramenta para gerenciamento facilitado do EKS
#### https://eksctl.io

#curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
#sudo mv /tmp/eksctl /usr/local/bin
#eksctl

#### Depois vamos instalar o awscli, a fim de acessar nossa conta na AWS

#curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
#unzip awscliv2.zip
#sudo ./aws/install

Após configurar o awscli com o aws configure, já é possível criar o cluster EKS

#eksctl create cluster --name=eks-cluster --version=1.24 --region=us-east-1 --nodegroup-name=eks-cluster-nodegroup --node-type=t2.micro --nodes=2 --nodes-min=1 --nodes-max=3 --managed

Caso o kubectl ainda não estiver instalado, será necessário rodar o seguinte comando, após a instalação da ferramenta:

#aws eks --region us-east-1 update-kubeconfig --name eks-cluster

Esses são os comandos para verificar o cluster

#kubectl get nodes -> Para verificar com o K8s
#eksctl get cluster -A -> Para verificar todos os clusters da conta
#eksctl get cluster -r us-east-1 -> Para verificar todos os clusters em uma região

Para aumentar o número de nodes:
#eksctl scale nodegroup --cluster=eks-cluster --nodes=3 --nodes-min=1 --nodes-max=3 --name=eks-cluster-nodegroup -r us-east-1

Para diminuir o número de nodes:
#eksctl scale nodegroup --cluster=eks-cluster --nodes=1 --nodes-min=1 --nodes-max=3 --name=eks-cluster-nodegroup -r us-east-1

Para deletar o cluster:
#eksctl delete cluster --name=eks-cluster -r us-east-1
#eksctl delete cluster --name=eks-cluster

#### Instalando o Kube-Prometheus

O primeiro passo é clonar o repositório

#git clone https://github.com/prometheus-operator/kube-prometheus

Após isso, devemos acessar o diretório, e rodar o setup, a fim de aplicar os manifestos

#cd kube-prometheus
#kubectl create -f manifests/setup


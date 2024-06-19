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

Vamos aplicar os manifestos.  Após isso, teremos o Prometheus e o AlertManager

#kubectl apply -f manifests/

O comando para verificar se os CRDs (Custom Resource Definitions) foram instalados é o seguinte:

#kubectl get servicemonitors -A

Já temos a Stack do Kube-Prometheus (Prometheus, Alertmanager, Blackbox Exporter e Grafana)

Verificando se tudo foi instalado

#kubectl get pods -n monitoring

#### Acessando o Grafana, o Prometheus e o Alertmanager

Vamos de port forward

##### Grafana
#kubectl port-forward -n monitoring svc/grafana 33000:3000
#http://localhost:33000 -> o acesso padrão é admin/admin

##### Prometheus
#kubectl port-forward -n monitoring svc/prometheus 39090:9090
#http://localhost:39090

##### Alertmanager
#kubectl port-forward -n monitoring svc/alertmanager 39093:9093
#http://localhost:39093

### Day-10

Criando Deploy e Service
Devemos criar um ConfigMap com as rotas /nginx_status para expor métricas do Nginx, e /metrics para expor o Nginx Exporter

  apiVersion: v1
  kind: ConfigMap
  metadata: 
    name: nginx-config
  data:
    nginx.conf: | # inicio da definição do arquivo de configuração do Nginx
      server {
        listen 80;
        location / {
          root /usr/share/nginx/html;
          index index.html index.htm;
        }
        location /metrics {
          stub_status on;
          access_log off;
        }
      }

#kubectl apply -f nginx-config.yaml
#kubectl get configmaps

Agora devemos criar a aplicação

apiVersion: apps/v1 # versão da API
kind: Deployment # tipo de recurso, no caso, um Deployment
metadata: # metadados do recurso 
  name: nginx-server # nome do recurso
spec: # especificação do recurso
  selector: # seletor para identificar os pods que serão gerenciados pelo deployment
    matchLabels: # labels que identificam os pods que serão gerenciados pelo deployment
      app: nginx # label que identifica o app que será gerenciado pelo deployment
  replicas: 3 # quantidade de réplicas do deployment
  template: # template do deployment
    metadata: # metadados do template
      labels: # labels do template
        app: nginx # label que identifica o app
      annotations: # annotations do template
        prometheus.io/scrape: 'true' # habilita o scraping do Prometheus
        prometheus.io/port: '9113' # porta do target
    spec: # especificação do template
      containers: # containers do template 
        - name: nginx # nome do container
          image: nginx # imagem do container do Nginx
          ports: # portas do container
            - containerPort: 80 # porta do container
              name: http # nome da porta
          volumeMounts: # volumes que serão montados no container
            - name: nginx-config # nome do volume
              mountPath: /etc/nginx/conf.d/default.conf # caminho de montagem do volume
              subPath: nginx.conf # subpath do volume
        - name: nginx-exporter # nome do container que será o exporter
          image: 'nginx/nginx-prometheus-exporter:0.11.0' # imagem do container do exporter
          args: # argumentos do container
            - '-nginx.scrape-uri=http://localhost/metrics' # argumento para definir a URI de scraping
          resources: # recursos do container
            limits: # limites de recursos
              memory: 128Mi # limite de memória
              cpu: 0.3 # limite de CPU
          ports: # portas do container
            - containerPort: 9113 # porta do container que será exposta
              name: metrics # nome da porta
      volumes: # volumes do template
        - configMap: # configmap do volume, nós iremos criar esse volume através de um configmap
            defaultMode: 420 # modo padrão do volume
            name: nginx-config # nome do configmap
          name: nginx-config # nome do volume

#kubectl apply -f nginx-deployment.yaml
#kubectl get po
#kubectl get deploy

Agora é criar o service

apiVersion: v1 # versão da API
kind: Service # tipo de recurso, no caso, um Service
metadata: # metadados do recurso
  name: nginx-svc # nome do recurso
  labels: # labels do recurso
    app: nginx # label para identificar o svc
spec: # especificação do recurso
  ports: # definição da porta do svc 
  - port: 9113 # porta do svc
    name: metrics # nome da porta
  selector: # seletor para identificar os pods/deployment que esse svc irá expor
    app: nginx # label que identifica o pod/deployment que será exposto

Para verificar se o Nginx está rodando
#curl http://<EXTERNAL-IP-DO-SERVICE>:80

Verificando as métricas:
#curl http://<EXTERNAL-IP-DO-SERVICE>:80/nginx_status
#curl http://<EXTERNAL-IP-DO-SERVICE>:80/metrics
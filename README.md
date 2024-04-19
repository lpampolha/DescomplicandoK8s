# DescomplicandoK8s
# Baixando a imagem do Debian

sudo debootstrap stable /debian http://deb.debian.org/debian

# Montando o container

unshare --mount --pid --net --uts --ipc --map-root-user --user --fork chroot /debian /bin/bash

# Montando diretórios 

mount -t proc none /proc/
ps -ef
ip a
mount -t sysfs none /sys/
mount -t tmpfs none /tmp/

# CGROUP

## Instalando pacotes
sudo apt-get install cgroup-tools

cgcreate -g cpu,memory,blkio,devices,freezer:giropops
ls /sys/fs/cgroup/cpu/giropops

### chamando novamente o container com o unshare, o PID do bash  fará referência ao do unshare, como sendo processo filho, no HOST.

### Aqui a gente associa os controles do container ao HOST, para melhor gerenciar o processo no cgroup que acabamos de montar
cgclassify -g cpu,memory,blkio,devices,freezer:giropops "PID bash"

### Para conferir
cat /sys/fs/cgroup/cpu/giropops/cgroup.procs

### Verificando o número de cpus
getconf _NPROCESSORS_ONLN (mesma informação dada com cat /proc/cpuinfo)

### Setando o limite de cpu do container
cgset -r cpu.cfs_quota_us=1000 giropops

### verificando se o setar de 1% funcionou
cat /sys/fs/cgroup/cpu/giropops/cpu.cfs_quota_us

### Testando
###sem o stress, usaremos o dd
dd if=/dev/zero of=catota.img bs=8k count=256k

#### Kubernetes
### Day-3
## Formas de pegar a saída e criar um deployment
k get deployments.apps nginx-deployment -o yaml > teste.yaml

k create deployment --image nginx --replicas 3 nginx-deployment --dry-run=client -o yaml > deployment.yaml

## Estratégias de atualizações nos Deployments (RollingUpdate e Recreate)
k rollout status deployment -n giropops nginx-deployment


# VOLTHA versão 2.9.9

ubuntu server 20.04 

4 CPU - 12 GB RAM - 30 GB Disk

(K8s Single node - Flannel CNI plugin)

## Pre requisitos
- cluster kubernetes;
- helm;

## Instalação de pré requisitos

### Install docker

```js
curl -fsSL https://get.docker.com -o get-docker.sh

sudo sh get-docker.sh

sudo usermod -aG docker $USER     

newgrp docker
```

### Install helm
```js
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3

chmod 700 get_helm.sh

./get_helm.sh
```

### Install kubernetes

```js
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add

sudo apt-add-repository "deb http://apt.kubernetes.io/ kubernetes-xenial main"

sudo apt-get install software-properties-common -y

sudo apt-get install kubeadm kubelet kubectl

sudo swapoff -a

sudo hostnamectl set-hostname master-node

rm /etc/containerd/config.toml

systemctl restart containerd

sudo kubeadm init --pod-network-cidr=10.244.0.0/16

mkdir -p $HOME/.kube

sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config

sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Flannel
```js
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml

```

Taint
```js
kubectl taint nodes --all node-role.kubernetes.io/master-
```

## Instalação do Voltha helm charts

### Install voltha infra
```js
helm repo add onf https://charts.opencord.org

helm repo update
```

```js
helm upgrade --install --create-namespace -n infra  --version 2.9.9 voltha-infra onf/voltha-infra

kubectl taint nodes --all node-role.kubernetes.io/control-plane:NoSchedule-
```


### Install voltha-stack
```js
helm upgrade --install --create-namespace   -n voltha voltha onf/voltha-stack   --version 2.9.9 --set global.stack_name=voltha   --set global.voltha_infra_name=voltha-infra   --set global.voltha_infra_namespace=infra
```

### Install BBSIM
```js
helm upgrade --install -n voltha bbsim0 onf/bbsim --set olt_id=10
```

### Port-forward 

```js
kubectl -n infra port-forward --address 0.0.0.0 svc/voltha-infra-onos-classic-hs 8101:8101  2>&1 >> port-forward.log &

kubectl -n infra port-forward --address 0.0.0.0 svc/voltha-infra-onos-classic-hs 8181:8181  2>&1 >> port-forward.log &

kubectl -n voltha port-forward --address 0.0.0.0 svc/voltha-voltha-api 55555 2>&1 >> port-forward.log &
```

`obs: o port-forward realizado dessa forma permite que eles fiquem sendo executados em background e gera um arquivo de log port-forward.log.`

### Installing and Configuring voltctl
```js
HOSTOS="$(uname -s | tr "[:upper:]" "[:lower:"])"
HOSTARCH="$(uname -m | tr "[:upper:]" "[:lower:"])"
if [ "$HOSTARCH" == "x86_64" ]; then
    HOSTARCH="amd64"
fi
sudo wget https://github.com/opencord/voltctl/releases/download/v1.3.1/voltctl-1.3.1-$HOSTOS-$HOSTARCH -O /usr/local/bin/voltctl

chmod +x /usr/local/bin/voltctl

source <(voltctl completion bash)
```

`OBS: o serviço do voltctl se conecta ao pod voltha-voltha-api (porta 55555), logo precisa que o port-forward para esse serviço seja executado para que ele funcione corretamente.`

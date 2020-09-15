# Single Control-Plane Kubernetes Cluster Setup


## Install kubeadm (all nodes)
```bash
user@master:~$ sudo apt-get update && sudo apt-get install -y apt-transport-https curl
user@master:~$ curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
user@master:~$ cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
user@master:~$ sudo apt-get update
user@master:~$ sudo apt-get install -y kubelet kubeadm kubectl
user@master:~$ sudo apt-mark hold kubelet kubeadm kubectl
```
## Install Docker (all nodes)
```bash
user@master:~$ sudo apt install -y docker.io
```

## Enable cgroups
```bash
user@master:~$ sudo docker info
user@master:~$ sudo vim /etc/docker/daemon.json
```
```json
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
```

## For the Raspberry Pi 4

Enable following options
* cgroup_enable=cpuset
* cgroup_enable=memory
* cgroup_memory=1
* swapaccount=1

```bash
user@master:~$ sudo sed -i '$ s/$/ cgroup_enable=cpuset cgroup_enable=memory cgroup_memory=1 swapaccount=1/' /boot/firmware/cmdline.txt
```
##### *** DO NOT FORGET TO REBOOT SYSTEM AFTER CHANGING CMDLINE.TXT ***


## Initialise Master Node

```bash
user@master:~$ sudo kubeadm init --pod-network-cidr=10.244.0.0/16
user@master:~$ mkdir -p $HOME/.kube
user@master:~$ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
user@master:~$ sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

## Install Flannel CNI (master only)

```bash
user@master:~$ kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```
Wait for master node to reach ready state
```bash
user@master:~$ kubectl get nodes
```

## Add Worker Nodes


```bash
# Token
user@master:~$ sudo kubeadm token list
```
```bash
# Hash
user@master:~$ openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | openssl dgst -sha256 -hex | sed 's/^.* //'
```
Join Node
```bash
user@slave:~$ sudo kubeadm join {MASTER_IP}:6443 --token {TOKEN} --discovery-token-ca-cert-hash sha256:{HASH}
```

### Check Node Status
```bash
user@master:~$ kubectl get nodes
```

---

## Install MetalLB

```bash
user@master:~$ kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.9.3/manifests/namespace.yaml
user@master:~$ kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.9.3/manifests/metallb.yaml
```

### Create MetalLB Secret

```bash
user@master:~$ kubectl create secret generic -n metallb-system memberlist --from-literal=secretkey="$(openssl rand -base64 128)"
```

### Configure MetalLB

```bash
# Create the ConfigMap
user@master:~$ cat <<EOF | kubectl create -f -
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: config
data:
  config: |
    address-pools:
    - name: address-pool-1
      protocol: layer2
      addresses:
      - 192.168.2.128-192.168.2.254
EOF
```

*** DO NOT FORGET TO SPECIFY CORRECT SUBNET ***




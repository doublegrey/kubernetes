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

---

## Keepalived Config

```au
global_defs {
   router_id KubernetesVIP
}

vrrp_instance APIServerVIP {
    interface ens192
    state MASTER
    priority 100
    mcast_src_ip 172.17.138.130
    virtual_router_id 61
    advert_int 1
    nopreempt
    authentication {
          auth_type PASS
          auth_pass PASSWORD
    }
    unicast_peer {
        172.17.138.131
        172.17.138.132
    }
    virtual_ipaddress {
        172.17.138.140
    }
    #track_script {
    #    APIServerProbe
    #}
}

#vrrp_script APIServerProbe {
#    # Health check the Kubernetes API Server
#    script "curl -k https://172.17.138.130:6443"
#    interval 3
#    timeout 9
#    fall 2
#    rise 2
#}
```

---

## Haproxy Config

```
global
    log /dev/log    local0
    log /dev/log    local1 notice
    chroot /var/lib/haproxy
    stats socket /run/haproxy/admin.sock mode 660 level admin expose-fd listeners
    stats timeout 30s
    user haproxy
    group haproxy
    daemon

    # Default SSL material locations
    ca-base /etc/ssl/certs
    crt-base /etc/ssl/private

    # See: https://ssl-config.mozilla.org/#server=haproxy&server-version=2.0.3&config=intermediate
    ssl-default-bind-ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384
    ssl-default-bind-ciphersuites TLS_AES_128_GCM_SHA256:TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256
    ssl-default-bind-options ssl-min-ver TLSv1.2 no-tls-tickets

defaults
    log     global
    mode    http
    option  httplog
    option  dontlognull
    timeout connect 5000
    timeout client  50000
    timeout server  50000
    errorfile 400 /etc/haproxy/errors/400.http
    errorfile 403 /etc/haproxy/errors/403.http
    errorfile 408 /etc/haproxy/errors/408.http
    errorfile 500 /etc/haproxy/errors/500.http
    errorfile 502 /etc/haproxy/errors/502.http
    errorfile 503 /etc/haproxy/errors/503.http
    errorfile 504 /etc/haproxy/errors/504.http


frontend http_stats
    bind *:8080
    mode http
    stats uri /haproxy?stats

frontend kube_api_server
    bind 0.0.0.0:6443
    mode tcp
    option tcplog
    timeout client  10800s
    default_backend controlPlanes

backend controlPlanes
    mode tcp
    option tcplog
    balance leastconn
    timeout server  10800s
    server controlPlane01 172.17.138.130:6444 check
    server controlPlane02 172.17.138.131:6444 check
    server controlPlane03 172.17.138.132:6444 check
```


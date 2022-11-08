# Setup Kubernetes on a NUC/VM

## Base

Standard minimal Ubuntu focal, but it will work on yammy

Patched to latest as at 07/11/2022

Standard access/login used in this example, change as necessary for your NUC

Nuc IP : 192.168.101.77
User: install
Pass: install

## Expectation

Clean Ubuntu Focal/Jammy
Internet Access

```
apt update
apt upgrade
```

## Start

### Requirements

2 extra LAN IP's, STATIC, to get traffic into the cluster, if you don't need for UDP, only one IP is required. Else you will need to use node ports, which deviates from how vraptor works.

### OS Level Depedencies

```
sudo apt-get install -y apt-transport-https ca-certificates curl
```

### Install container runtime - containerd

Cannot co-exist with docker!

```
curl -fsSLo containerd-config.toml \
  https://gist.githubusercontent.com/oradwell/31ef858de3ca43addef68ff971f459c2/raw/5099df007eb717a11825c3890a0517892fa12dbf/containerd-config.toml
sudo mkdir /etc/containerd
sudo mv containerd-config.toml /etc/containerd/config.toml
```

```
curl -fsSLo containerd-1.5.13-linux-amd64.tar.gz https://github.com/containerd/containerd/releases/download/v1.5.13/containerd-1.5.13-linux-amd64.tar.gz
```

```
sudo tar Cxzvf /usr/local containerd-1.5.13-linux-amd64.tar.gz
```

```
sudo curl -fsSLo /etc/systemd/system/containerd.service https://raw.githubusercontent.com/containerd/containerd/main/containerd.service
```

```
sudo systemctl daemon-reload
sudo systemctl enable --now containerd
```

### Runc

```
curl -fsSLo runc.amd64 https://github.com/opencontainers/runc/releases/download/v1.1.3/runc.amd64
```

```
sudo install -m 755 runc.amd64 /usr/local/sbin/runc
```

### Networking Plugins

```
curl -fsSLo cni-plugins-linux-amd64-v1.1.1.tgz https://github.com/containernetworking/plugins/releases/download/v1.1.1/cni-plugins-linux-amd64-v1.1.1.tgz
```

```
sudo mkdir -p /opt/cni/bin
sudo tar Cxzvf /opt/cni/bin cni-plugins-linux-amd64-v1.1.1.tgz
```

### Enable low-level bridge networking

```
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
```

```
sudo modprobe -a overlay br_netfilter
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
```

```
sudo sysctl --system
```

### Add K8s stack to manage the containerd/cni/runc

#### Add Kubernetes GPG key

```
sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
```

#### Add Kubernetes apt repository

```
echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

```
sudo apt-get update
```

### Install k8s tooling

#### Install at iotnxt supported versions

```
sudo apt-get install -y kubelet=1.21.14-00 kubeadm=1.21.14-00 kubectl=1.21.14-00
```

#### Lock the package versions so it doesn't auto upgrade

```
sudo apt-mark hold kubelet kubeadm kubectl
```

### K8s doesn't work with swap, disable it

```
sudo swapoff -a
sudo sed -i -e '/swap/d' /etc/fstab
```

### Setup a cluster

You need to decide a pod network, decide it here BEFORE you do anything else, it must be outside any network to be used in future, should be private. If you change it away from 192.168.0.0/16, you need to update the calico settings to match this

```
sudo kubeadm init --pod-network-cidr=192.168.0.0/16
```

### Link up kubectl to your user

```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

#### Test

```
kubectl get nodes
```

Should output a node, likely unready (needs networking)

```
install@nucemu:~$ sudo kubectl get node
NAME     STATUS     ROLES                  AGE     VERSION
nucemu   NotReady   control-plane,master   4m43s   v1.21.14
```

### Convert to single node cluster, so it can run vraptor pods as a single thing

```
kubectl taint nodes --all node-role.kubernetes.io/master-
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
```

### Install Helm

```
curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash
```

### Install Calico

```
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.24.4/manifests/tigera-operator.yaml
```

```
curl https://raw.githubusercontent.com/projectcalico/calico/v3.24.4/manifests/custom-resources.yaml -O
```

If you changed the pod network above, edit this file first and change the ipPools - cidr to be in the same range.

```
vi custom-resources.yaml
```

```
kubectl create -f custom-resources.yaml
```

Calico will start firing up and create itself, which should turn the node ready in a few minutes

Verify the node goes ready

```
kubectl get nodes
```

Must appear ready

```
root@nucemu:/home/install# kubectl get nodes
NAME     STATUS   ROLES                  AGE   VERSION
nucemu   Ready    control-plane,master   66m   v1.21.14
```

At this point the pods on your cluster should look like this

```
kubectl get pods -A

NAMESPACE          NAME                                       READY   STATUS    RESTARTS   AGE
calico-apiserver   calico-apiserver-c47c9fbcb-2p9p2           1/1     Running   0          2m2s
calico-apiserver   calico-apiserver-c47c9fbcb-cfdkk           1/1     Running   0          2m2s
calico-system      calico-kube-controllers-55dc8fbb6d-vwrf5   1/1     Running   0          3m49s
calico-system      calico-node-5np94                          1/1     Running   0          3m50s
calico-system      calico-typha-6b988667c5-kvttx              1/1     Running   0          3m50s
calico-system      csi-node-driver-gh6cs                      2/2     Running   0          2m42s
kube-system        coredns-558bd4d5db-dr4kx                   1/1     Running   0          45m
kube-system        coredns-558bd4d5db-ssftx                   1/1     Running   0          45m
kube-system        etcd-nucemu                                1/1     Running   0          45m
kube-system        kube-apiserver-nucemu                      1/1     Running   0          45m
kube-system        kube-controller-manager-nucemu             1/1     Running   0          45m
kube-system        kube-proxy-96l64                           1/1     Running   0          45m
kube-system        kube-scheduler-nucemu                      1/1     Running   0          45m
tigera-operator    tigera-operator-7f589df4bf-hpgvj           1/1     Running   0          43m
```

Congrats, you have a kubernetes cluster running!

You can add it into rancher here already if you want.

### Install MetalLB

To get traffic into the cluster, you need a loadbalancer, for on-prem, we use MetalLB

Requirement : Extra IP on the LAN, for each protocol... TCP -> One, UDP -> One

#### Enable strict ARP to bond IP's that the cluster will respond to

```
kubectl get configmap kube-proxy -n kube-system -o yaml | \
sed -e "s/strictARP: false/strictARP: true/" | \
kubectl apply -f - -n kube-system
```

#### Install MetalLB via Helm

```
helm repo add metallb https://metallb.github.io/metallb
helm -n metallb-system install metallb metallb/metallb --create-namespace
```

#### Configure ip's for metallb use (expose the vraptor)

Specify the IP block/ip's for metallb to use for exposing pods

```
cat <<EOF | sudo tee metallb-config.yaml
apiVersion: metallb.io/v1beta2
kind: IPAddressPool
metadata:
  name: vraptor-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.101.80-192.168.101.81
EOF
```

```
kubectl -n metallb-system apply -f metallb-config.yaml
```

#### Verify

your status should now be

```
NAMESPACE          NAME                                       READY   STATUS    RESTARTS   AGE
calico-apiserver   calico-apiserver-c47c9fbcb-2p9p2           1/1     Running   0          9m41s
calico-apiserver   calico-apiserver-c47c9fbcb-cfdkk           1/1     Running   0          9m41s
calico-system      calico-kube-controllers-55dc8fbb6d-vwrf5   1/1     Running   0          11m
calico-system      calico-node-5np94                          1/1     Running   0          11m
calico-system      calico-typha-6b988667c5-kvttx              1/1     Running   0          11m
calico-system      csi-node-driver-gh6cs                      2/2     Running   0          10m
kube-system        coredns-558bd4d5db-dr4kx                   1/1     Running   0          53m
kube-system        coredns-558bd4d5db-ssftx                   1/1     Running   0          53m
kube-system        etcd-nucemu                                1/1     Running   0          53m
kube-system        kube-apiserver-nucemu                      1/1     Running   0          53m
kube-system        kube-controller-manager-nucemu             1/1     Running   0          53m
kube-system        kube-proxy-96l64                           1/1     Running   0          53m
kube-system        kube-scheduler-nucemu                      1/1     Running   0          53m
metallb-system     metallb-controller-5b98846878-cgdmw        1/1     Running   0          33s
metallb-system     metallb-speaker-jvgpv                      1/1     Running   0          33s
tigera-operator    tigera-operator-7f589df4bf-hpgvj           1/1     Running   0          51m
```

### Ready for vraptor deployment and adding to rancher

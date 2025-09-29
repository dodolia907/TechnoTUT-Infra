# Kubernetes (k8s) Setup
Special thanks to [@launchpencil](https://github.com/launchpencil) for guiding us through our introduction to Kubernetes.

## Environment
Operating System: Debian 13 trixie  
Nodes: 3 (pjsekai, maimai, chunithm)

| Node | CPU | Memory | Disk | IP Address | Role |
|------|-----|--------|------|------------|------|
| pjsekai | 4CPU | 8GB | 500GB HDD (Hitachi) | 192.168.30.200/24 | Control-Plane |
| maimai | 2CPU | 12GB | 320GB HDD (Fujitsu) | 192.168.30.201/24 | Worker |
| chunithm | 2CPU | 12GB | 500GB HDD (WD Blue) | 192.168.30.202/24 | Worker |

## Install k8s
Prepare Container Runtime at all nodes. We use CRI-O.  
https://kubernetes.io/ja/docs/setup/production-environment/tools/kubeadm/install-kubeadm/  
https://kubernetes.io/ja/docs/setup/production-environment/container-runtimes/  
https://github.com/cri-o/packaging/blob/main/README.md#usage
```bash
## Enable IPv4 forwarding and bridge network settings
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system

## Login as root
sudo su -

## Install prerequisites
apt-get update
apt-get install -y curl gpg

## Version variables
KUBERNETES_VERSION=v1.34
CRIO_VERSION=v1.34

## Add the Kubernetes repository
curl -fsSL https://pkgs.k8s.io/core:/stable:/$KUBERNETES_VERSION/deb/Release.key |
    gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/$KUBERNETES_VERSION/deb/ /" |
    tee /etc/apt/sources.list.d/kubernetes.list

## Add the CRI-O repository
curl -fsSL https://download.opensuse.org/repositories/isv:/cri-o:/stable:/$CRIO_VERSION/deb/Release.key |
    gpg --dearmor -o /etc/apt/keyrings/cri-o-apt-keyring.gpg

echo "deb [signed-by=/etc/apt/keyrings/cri-o-apt-keyring.gpg] https://download.opensuse.org/repositories/isv:/cri-o:/stable:/$CRIO_VERSION/deb/ /" |
    tee /etc/apt/sources.list.d/cri-o.list

## Install kubeadm, kubelet, kubectl, cri-o
apt-get update
apt-get install -y kubelet kubeadm kubectl cri-o

## Start CRI-O
systemctl enable --now crio.service
```

## Create Cluster
At Control-Plane node (pjsekai)
```bash
kubeadm init --pod-network-cidr=10.10.0.0/16
```
Copy the kubeadm join command outputted at the end of the initialization process, you will need it to join worker nodes to the cluster.

At Worker nodes (maimai, chunithm)
```bash
kubeadm join ... (the command you copied from the control-plane node)
```

## Configure kubectl
At Control-Plane node (pjsekai)
```bash
## Login as normal user
## Copy kubeconfig file to user's home directory
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

## Network settings
Kubernetes network is implemented by CNI (Container Network Interface) plugins. We use Calico. Calico provides pure L3 network.  

Install Calico at Control-Plane node (pjsekai)  
https://docs.tigera.io/calico/latest/getting-started/kubernetes/self-managed-onprem/onpremises  
```bash
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.30.3/manifests/operator-crds.yaml
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.30.3/manifests/tigera-operator.yaml
```
Download custom Calico configuration
```bash
curl https://raw.githubusercontent.com/projectcalico/calico/v3.30.3/manifests/custom-resources.yaml -O
```
Edit `custom-resources.yaml`. [Sample](./setup/calico-manifest/custom-resources.yaml)  

Apply [custom-resources.yaml](./setup/calico-manifest/custom-resources.yaml)
```bash
kubectl create -f custom-resources.yaml
```

Verify Calico installation
```bash
watch kubectl get pods -A -o wide
```

Install Calicoctl
```bash
curl -L https://github.com/projectcalico/calico/releases/download/v3.30.3/calicoctl-linux-amd64 -o calicoctl
chmod +x ./calicoctl
sudo mv ./calicoctl /usr/local/bin/calicoctl
```

Configure BGP Peering as ToR  
https://docs.tigera.io/calico/latest/networking/configuring/bgp

Create `bgppeer.yaml`. [Sample](./setup/calico-manifest/bgppeer.yaml)  
Create `bgpconfig.yaml`. [Sample](./setup/calico-manifest/bgpconfig.yaml)  

```bash
calicoctl apply -f bgppeer.yaml
calicoctl apply -f bgpconfig.yaml
```

Configure Router
```
## NEC UNIVERGE IX Series
router bgp 64646
  neighbor 192.168.30.200 remote-as 64646
  neighbor 192.168.30.201 remote-as 64646
  neighbor 192.168.30.202 remote-as 64646
  exit

write memory
show ip bgp summary
```
Router says "ESTABLISHED", It works!

Confirm network settings
```bash
calicoctl get nodes -o wide
calicoctl get bgpPeer -o wide
calicoctl get ippool -o wide
calicoctl get bgpConfiguration -o wide
watch kubectl get pod -A -o wide
ip route
```

## Setup Persistent Volume using nfs-subdir-external-provisioner
nfs-subdir-external-provisioner: https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner  
Install nfs server
```bash
sudo apt -y install nfs-kernel-server
sudo mkdir /nfs
sudo systemctl enable --now nfs-server
sudo vim /etc/exports
```
``` /etc/exports
/nfs 192.168.30.0/24(rw,no_root_squash)
```
```bash
sudo systemctl restart nfs-kernel-server
```

All nodes require nfs-common package.
At all nodes:
```bash
sudo apt -y install nfs-common
```

Install Helm (Kubernetes package manager)
```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```
Install nfs-subdir-external-provisioner
```bash
helm repo add nfs-subdir-external-provisioner https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/
helm install nfs-subdir-external-provisioner nfs-subdir-external-provisioner/nfs-subdir-external-provisioner --set nfs.server=192.168.30.200 --set nfs.path=/nfs --namespace nfs-provisioner --create-namespace
```

## Setup MetalLB
https://metallb.universe.tf/installation/
```bash
## Enable Strict ARP
## see what changes would be made, returns nonzero returncode if different
kubectl get configmap kube-proxy -n kube-system -o yaml | \
sed -e "s/strictARP: false/strictARP: true/" | \
kubectl diff -f - -n kube-system

## actually apply the changes, returns nonzero returncode on errors only
kubectl get configmap kube-proxy -n kube-system -o yaml | \
sed -e "s/strictARP: false/strictARP: true/" | \
kubectl apply -f - -n kube-system

## Install MetalLB
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.15.2/config/manifests/metallb-native.yaml
```
Create IPAddressPool  
https://metallb.universe.tf/configuration/  
Sample: [metallb-ipaddresspool.yaml](./setup/metallb/metallb-ipaddresspool.yaml)
```bash
kubectl apply -f metallb-ipaddresspool.yaml
```

## Install ArgoCD
```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
curl -L https://github.com/argoproj/argo-cd/releases/download/v3.1.7/argocd-linux-amd64 -o argocd
chmod +x argocd
sudo mv argocd /usr/local/bin/argocd
argocd admin initial-password -n argocd
argocd login 10.11.0.0
argocd account update-password
```
Access `10.11.0.0` and login `admin` with password

## if you want to reset
Uninstall and retry install  
https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/#remove-the-node  

at Control-Plane node:
```bash
kubectl drain <node name> --delete-emptydir-data --force --ignore-daemonsets
```

at all nodes:
```bash
kubeadm reset
iptables -F && iptables -t nat -F && iptables -t mangle -F && iptables -X
ipvsadm -C
kubectl delete node <node name>
```

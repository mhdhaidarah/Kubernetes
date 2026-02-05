This is tested on ubuntu to create Kubernetes Cluster Include any number of Masters and Workers

## This commands must be run at root privilege
```bash
sudo su
```

## All Nodes set Hostname and update
```bash
apt update && apt upgrade -y
hostnamectl set-hostname (CHANGE HOSTNAME)
```

## All Nodes
```bash
swapoff -a
sed -i '/ swap / s/^/#/' /etc/fstab

cat <<EOF | tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

modprobe overlay
modprobe br_netfilter


cat <<EOF | tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sysctl --system
```

## All Nodes
```bash
apt install -y containerd
```

## All Nodes (CHANGE) -> SystemdCgroup = true
```bash
mkdir -p /etc/containerd
containerd config default > /etc/containerd/config.toml
nano /etc/containerd/config.toml
```

## All Nodes
```bash
systemctl restart containerd
systemctl enable containerd
apt install -y apt-transport-https ca-certificates curl
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key \
| gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] \
https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /" \
| tee /etc/apt/sources.list.d/kubernetes.list
apt update
apt install -y kubelet kubeadm kubectl
apt-mark hold kubelet kubeadm kubectl
```

## Master Only
```bash
ctr image pull ghcr.io/kube-vip/kube-vip:v0.7.1
mkdir -p /etc/kubernetes/manifests
```
## Master Only Change IP and Interface Name
```bash
ctr run --rm --net-host ghcr.io/kube-vip/kube-vip:v0.7.1 vip \
/kube-vip manifest pod \
--interface ens33 \
--address 192.168.1.100 \
--controlplane \
--services \
--leaderElection \
> /etc/kubernetes/manifests/kube-vip.yaml
```

## Master Only
```bash
kubeadm init \
--control-plane-endpoint "192.168.1.100:6443" \
--upload-certs \
--pod-network-cidr=10.244.0.0/16
```

## Save the join commands shown at the end of the command 

## Master Only
mkdir -p $HOME/.kube
cp /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config

## Example join commands
```bash
## All Masters Join
kubeadm join 192.168.1.100:6443 \
--token <token> \
--discovery-token-ca-cert-hash sha256:<hash> \
--control-plane --certificate-key <cert-key>


## All Workers Join
kubeadm join 192.168.1.100:6443 \
--token <token> \
--discovery-token-ca-cert-hash sha256:<hash>
```

## All Nodes
```bash
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```
## Testing
```bash
kubectl create deployment nginx --image=nginx --replicas=10
kubectl expose deployment nginx \
--type=NodePort \
--port=8080 \
--target-port=80
kubectl get svc
```

## Show info about cluster
```bash
kubectl get nodes
```
```bash
kubectl get pods -o wide
```








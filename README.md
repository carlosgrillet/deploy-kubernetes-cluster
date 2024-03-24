### First steps

> Hardware requirements: Ubuntu 23.10, Minimum 2GB RAM, Minimum 2 CPU core.

- Set server hostname and add nodes hostnames to /etc/hosts file
- Make sure nodes and master can communicate each others  
- Nodes and Controller should be able to ping 8.8.8.8 or reach internet

### Proxy configuration (if needed)

> For apt
```bash
$ cat <<EOF > /etc/apt/apt.conf
Acquire::http::Proxy "http://PROXY:PORT";
Acquire::https::Proxy "https://PROXY:PORT";
EOF
```

> For systemd
```bash
sudo systemctl set-environment HTTP_PROXY=http://PROXY:PORT
sudo systemctl set-environment HTTPS_PROXY=http://PROXY:PORT
sudo systemctl set-environment NO_PROXY=<kube-net>/<prefix>
sudo systemctl restart containerd.service
```

### Add IP and hostname of cluster ndoes to hosts file

```
#k8s-cluster
<master-ip> <master-hostname>
<node-ip> <node-hostname>
<node-ip> <node-hostname>
<node-ip> <node-hostname>
```

### Disable swap before installing kuerbnetes

```bash
sudo swapon --show  
sudo swapoff -a  
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```
  
### Load kernel modules

```bash
sudo tee /etc/modules-load.d/containerd.conf <<EOF  
overlay  
br_netfilter  
EOF
```

```bash
sudo modprobe overlay  
sudo modprobe br_netfilter
```

### Configure networking for kubernetes  

```bash
sudo tee /etc/sysctl.d/kubernetes.conf <<EOF  
net.bridge.bridge-nf-call-ip6tables = 1  
net.bridge.bridge-nf-call-iptables = 1  
net.ipv4.ip_forward = 1  
EOF
```

```bash
sudo sysctl --system
```

### Enable ipv4 forwarding

```bash
echo 1 > /proc/sys/net/ipv4/ip_forward
```

### Install needed packages

```bash
sudo apt install -y curl gnupg2 software-properties-common apt-transport-https ca-certificates
```

### Add docker repo and gpg key

```bash
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmour -o /etc/apt/trusted.gpg.d/docker.gpg
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"  
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu  jammy stable"
sudo apt update
```

### Install containerd runtime

```bash
sudo apt install -y containerd.io  
containerd config default | sudo tee /etc/containerd/config.toml >/dev/null 2>&1  
sudo sed -i 's/SystemdCgroup \= false/SystemdCgroup \= true/g' /etc/containerd/config.toml
```

```bash
sudo systemctl restart containerd  
sudo systemctl enable containerd
```

### Add kubernetes repo and gpg key

```bash
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

### Install kubeadm, kubectl & kubelet

```bash
sudo apt update  
sudo apt install -y kubelet kubeadm kubectl  
sudo apt-mark hold kubelet kubeadm kubectl
```

### Start kubadm in control-plane node

```bash
sudo kubeadm init --control-plane-endpoint=<master-hostname>
```

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### apply Calico network manifest

```bash
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.2/manifests/calico.yaml
```

## Join a new node to the cluster  

> In the control-plane node run this command
```bash
kubeadm token create --print-join-command
```
> Now copy the output and paste it in the different nodes

```bash
kubeadm join k8s-control:6443 --token <TOKEN> --discovery-token-ca-cert-hash sha256:<HASH>
```

### Install kubecolor tool (Optional)

```bash
wget https://github.com/hidetatz/kubecolor/releases/download/v0.0.25/kubecolor_0.0.25_Linux_arm64.tar.gz
tar -xvf ubecolor_0.0.25_Linux_arm64.tar.gz
sudo mv kubecolor /usr/bin
```

> Now you can create an alias for kubecolor or use the tool like `kubecolor get nodes`

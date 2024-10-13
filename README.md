## Kubernetes Certification Practical Guides

Best way of getting certified is by learning by doing.

Inside this repository, you will many kubernetes examples, exercises and quizzes.

## Kubernetes Certification Curricaulam

https://github.com/cncf/curriculum

# Setup

#### Disable firewall

```bash
sudo systemctl stop ufw
sudo ufw disable
sudo systemctl status ufw
```

#### Update and upgrade the system / CP-Worker

```bash
sudo apt update && sudo apt upgrade -y
```

#### Enable time-sync with an NTP server / CP-W

```bash
sudo apt install systemd-timesyncd
sudo timedatectl set-ntp true
sudo timedatectl status
sudo timedatectl set-timezone UTC
```

#### Turn of the swap / CP-W

```bash
sudo swapoff -a
sudo sed -i.bak -r 's/(.+ swap .+)/#\1/' /etc/fstab

free -m

cat /etc/fstab | grep swap
```

#### Configure required kernel modules

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

#TODO sysctl params required by setup, params persist across reboots
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# Apply sysctl params without reboot
sudo sysctl --system
```

#### Installing kubeadm, kubelet and kubectl

```bash
sudo apt-get update
# apt-transport-https may be a dummy package; if so, you can skip that package
sudo apt-get install -y apt-transport-https ca-certificates curl gpg

# If the directory `/etc/apt/keyrings` does not exist, it should be created before the curl command, read the note below.
sudo mkdir -p -m 755 /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

# This overwrites any existing configuration in /etc/apt/sources.list.d/kubernetes.list
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list


sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl

sudo systemctl enable --now kubelet
```

#### Add containerd repository key and add the repository to source list / CP-W

```bash
# Add Docker's official GPG key:
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null


sudo apt update
sudo apt install -y containerd.io

# sudo apt-get install docker-ce docker-ce-cli docker-buildx-plugin docker-compose-plugin (optional)


# add user to docker group
sudo groupadd docker
sudo usermod -aG docker $USER
newgrp docker
groups $USER
```

#### Configure containerd / CP-W

```bash
sudo mkdir -p /etc/containerd
```

_Generate the default config toml file_

```bash
sudo containerd config default | sudo tee /etc/containerd/config.toml
```

_Open the generated file in any text editor and verify whether the following settings are present. If not, you should set them as shown below_

```bash
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
  runtime_type = "io.containerd.runc.v2"  # <- note this, this line might have been missed
  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
    SystemdCgroup = true # <- note this, this could be set as false in the default configuration, please make it true

# run bash  sudo sed -i 's/            SystemdCgroup = false/            SystemdCgroup = true/' /etc/containerd/config.toml
```

#### Restart containerd

```bash
sudo systemctl restart containerd
sudo systemctl enable containerd
systemctl status containerd
```

#### Enable kubelet service / CP-W

```bash
sudo systemctl enable kubelet
sudo apt-get update
sudo apt-get install socat -y
```

#### Init kubeadm use calico network (only cp)

```bash
sudo kubeadm init --apiserver-advertise-address=$HOST_IP --control-plane-endpoint=$HOST_IP_CP:6443 --pod-network-cidr=192.168.0.0/16
# apply resource network with calico
sudo kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/$VERSION/manifests/tigera-operator.yaml

curl https://raw.githubusercontent.com/projectcalico/calico/$VERSION/manifests/custom-resources.yaml -O

kubectl create -f custom-resources.yaml
```

#### JOIN WORKER NODE TO CP (only W)

```bash
sudo kubeadm token create --print-join-command # run only master
# kubeadm join 192.168.10.100:6443 --token duc9o5.5p7eczpa6evbwqcb --discovery-token-ca-cert-hash sha256:0509dc2de0b303b6410bfcb56347364a03ef425cc49574124f4c79e50bb8ab49
sudo kubeadm join $HOST_CP --token $TOKEN --discovery-token-ca-cert-hash $SHA_HASH
```

#### Config use kubectl cluster in local (in local)

```bash
ssh -i ~/.ssh/k8s/id_rsa ci@192.168.10.100 'sudo cat /etc/kubernetes/admin.conf' > ~/.kube/config-mycluster
export KUBECONFIG=~/.kube/config:~/.kube/config-mycluster
kubectl config view --flatten > ~/.kube/config_temp
mv ~/.kube/config_temp ~/.kube/config
kubectl config view
kubectl config get-contexts
kubectl config use-context kubernetes-admin@kubernetes
```

#### reset kubeadm (only cp)

```bash
sudo kubeadm reset -f
sudo rm -rf $HOME/.kube
sudo rm -rf /etc/kubernetes
sudo rm -rf /var/lib/etcd
```

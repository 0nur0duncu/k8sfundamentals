# Kurulum
**Kubernetes Kurulum** konusuyla ilgili dosyalara buradan erişebilirsiniz.

## kubeadm kurulum

https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/

**0:** sanal makine oluşturma
```
$ multipass launch --name master -c 2 -m 2G -d 10G
$ multipass launch --name node1 -c 2 -m 2G -d 10G
```
* Master node'a bağlanıp ```sudo hostnamectl set-hostname master``` komutunu girin. node1'e bağlanıp ```sudo hostnamectl set-hostname node1``` komutunu girin.

**1:** Kernel modulleri aktive ediyor ve swap kapatıyoruz

```
$ sudo modprobe overlay
$ sudo modprobe br_netfilter
```

```
$ cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
```

```
$ cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
```

```
$ sudo sysctl --system
```

```
$ sudo swapoff -a
$ free -m
$ sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

**2:** containerd kurulumu

```
$ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker.gpg
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
$ sudo apt update
$ sudo apt install containerd.io
$ sudo systemctl daemon-reload
$ sudo systemctl enable --now containerd
$ sudo systemctl start containerd
$ sudo mkdir -p /etc/containerd
$ sudo su -
$ containerd config default | tee /etc/containerd/config.toml
$ exit
$ sudo sed -i 's/            SystemdCgroup = false/            SystemdCgroup = true/' /etc/containerd/config.toml
$ sudo systemctl restart containerd

```

**3:** kubeadm kurulumu


```
$ sudo ufw allow 6443/tcp
$ sudo ufw allow 2379:2380/tcp
$ sudo ufw allow 10250/tcp
$ sudo ufw allow 10259/tcp
$ sudo ufw allow 10257/tcp
$ sudo apt-get update
$ sudo apt-get install -y apt-transport-https ca-certificates curl gnupg2 software-properties-common conntrack iproute2
$ sudo mkdir -p -m 755 /etc/apt/keyrings
### Eğer Kubernetes 1.31'den başka bir versiyon yüklemek isterseniz aşağıdaki iki komuttaki v1.31 kısımlarını düzeltin ####
$ curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
$ echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
$ sudo apt-get update
$ sudo apt-get install -y kubelet kubeadm kubectl
$ sudo apt-mark hold kubelet kubeadm kubectl
```

**4:** kubernetes cluster kurulumu

```
$ sudo kubeadm config images pull
# ya da birden fazla CRI soket varsa
$ sudo kubeadm config images pull --cri-socket unix:///run/containerd/containerd.sock

$ sudo kubeadm init --pod-network-cidr=192.168.0.0/16 --apiserver-advertise-address=<ip> --control-plane-endpoint=<ip>
```
```
$ sudo vim /etc/hosts
   IP_Addreses         Dns_Name  

  127.0.0.1 localhost
  
  <ip> kubernetes.dev.env.test
  <ip> k8s-master-1

$ sudo vim /etc/systemd/resolved.conf
  [Resolve]
  DNS=8.8.8.8 1.1.1.1

$ sudo systemctl restart systemd-resolved
$ sudo kubeadm init --pod-network-cidr=10.244.0.0/16

# Create cluster
$ sudo kubeadm init --control-plane-endpoint="kubernetes.dev.env.test:6443" --apiserver-advertise-address=<ip> --node-name k8s-master-1 --pod-network-cidr=192.168.0.0/16
```

```
# Sorun olması durumunda sıfırlamak için
$ sudo kubeadm reset -f
# Kalan static pod manifestlerini sil
$ sudo rm -rf /etc/kubernetes/manifests
$ sudo rm -rf /etc/kubernetes/pki

# CNI ve kubelet kalıntılarını temizle
$ sudo rm -rf /etc/cni/net.d
$ sudo rm -rf /var/lib/kubelet/*

# Çalışan kubelet’i yeniden başlat
$ sudo systemctl restart containerd
$ sudo systemctl restart kubelet
```

```
$ mkdir -p $HOME/.kube
$ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
$ sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

```
$ sudo apt-get install curl gpg apt-transport-https --yes
$ curl -fsSL https://packages.buildkite.com/helm-linux/helm-debian/gpgkey | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
$ echo "deb [signed-by=/usr/share/keyrings/helm.gpg] https://packages.buildkite.com/helm-linux/helm-debian/any/ any main" | $ sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
$ sudo apt-get update
$ sudo apt-get install helm
```

```
$ kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.31.3/manifests/tigera-operator.yaml
$ kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.31.3/manifests/custom-resources.yaml
```

```
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
kubectl taint nodes --all node-role.kubernetes.io/master-
```

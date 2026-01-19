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
$ kubectl taint nodes --all node-role.kubernetes.io/control-plane-
$ kubectl taint nodes --all node-role.kubernetes.io/master-
```

```
# Metrics Server (HPA/kubectl top için)
$ kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

```
# Rancher kurulumu
$ kubectl apply -f https://github.com/cert-manager/cert-manager/releases/latest/download/cert-manager.yaml
$ kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/cloud/deploy.yaml
$ kubectl get pods -n ingress-nginx
$ kubectl get pods -n cert-manager
$ helm repo add rancher-latest https://releases.rancher.com/server-charts/latest
$ helm repo update
$ kubectl create namespace cattle-system
$ helm install rancher rancher-latest/rancher \
--namespace cattle-system \
--set hostname=rancher.<domain> \
--set replicas=1
$ kubectl -n cattle-system rollout status deploy/rancher
$ kubectl get secret --namespace cattle-system bootstrap-secret -o go-template='{{.data.bootstrapPassword|base64decode}}{{ "\n" }}'
```

```
$ kubectl apply -f - <<EOF
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: your-email@example.com
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - http01:
        ingress:
          class: nginx
EOF

$ kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/main/config/manifests/metallb-native.yaml
$ kubectl apply -f - <<EOF
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: default-pool
  namespace: metallb-system
spec:
  addresses:
  - 5.178.111.179/32
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: l2
  namespace: metallb-system
EOF
# yüklü nginx-ingress varsa
$ kubectl delete svc ingress-nginx-controller -n ingress-nginx
$ kubectl get svc -n ingress-nginx

$ kubectl -n ingress-nginx scale deployment ingress-nginx-controller --replicas=2
$ kubectl get pods -n ingress-nginx -o wide

# Rancher Ingress
$ kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: rancher
  namespace: cattle-system
  annotations:
    kubernetes.io/ingress.class: nginx
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - rancher.<domain>
      secretName: tls-rancher-ingress
  rules:
    - host: rancher.<domain>
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: rancher
                port:
                  number: 80
EOF


$ kubectl get ingress -n cattle-system
```

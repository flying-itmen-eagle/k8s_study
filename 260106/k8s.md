# Ubuntu 24.04 安裝 Docker 與 Kubernetes（k8s）完整教學

本教學適用於 **Ubuntu 24.04 LTS (Noble)**，內容涵蓋：

* Docker 安裝與驗證
* Kubernetes（k8s）官方新套件庫（pkgs.k8s.io）
* kubeadm 單節點學習環境初始化

適合對象：

* 正在學習 Kubernetes 的初學者
* 需要在本機 / VM 建立測試環境的人

---

## 一、環境需求

* OS：Ubuntu 24.04 LTS
* CPU：2 Core 以上（建議）
* RAM：4GB 以上（最低 2GB）
* 使用者具備 sudo 權限

更新系統：

```bash
sudo apt update && sudo apt upgrade -y
```

---

## 二、安裝 Docker（Container Runtime）

### 1️⃣ 移除舊版（若有）

```bash
sudo apt remove -y docker docker-engine docker.io containerd runc
```

### 2️⃣ 安裝必要套件

```bash
sudo apt install -y ca-certificates curl gnupg
```

### 3️⃣ 加入 Docker 官方 GPG key（新式 keyrings）

```bash
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
| sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
```

### 4️⃣ 設定 Docker APT repository

```bash
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
https://download.docker.com/linux/ubuntu noble stable" \
| sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

### 5️⃣ 安裝 Docker

```bash
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

### 6️⃣ 驗證 Docker

```bash
sudo docker run hello-world
```

### 7️⃣（可選）免 sudo 使用 docker

```bash
sudo usermod -aG docker $USER
newgrp docker
```

---

## 三、安裝 Kubernetes（kubeadm / kubelet / kubectl）

⚠️ **注意**：舊的 `apt.kubernetes.io (kubernetes-xenial)` 已停用，Ubuntu 24.04 必須使用 **pkgs.k8s.io**

### 1️⃣ 關閉 Swap（k8s 必要）

```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab
```

### 2️⃣ 載入核心模組與 sysctl 設定

```bash
sudo modprobe overlay
sudo modprobe br_netfilter

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system
```

### 3️⃣ 加入 Kubernetes GPG key（以 v1.29 為例）

```bash
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key \
| sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```

### 4️⃣ 設定 Kubernetes APT repository

```bash
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] \
https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /" \
| sudo tee /etc/apt/sources.list.d/kubernetes.list
```

### 5️⃣ 安裝 Kubernetes 元件

```bash
sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

確認版本：

```bash
kubectl version --client
kubeadm version
```

---

## 四、初始化 Kubernetes（單節點教學環境）

### 1️⃣ 初始化 control-plane

```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
```

完成後會看到類似訊息：

```text
Your Kubernetes control-plane has initialized successfully!
```

### 2️⃣ 設定 kubectl 使用者設定

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### 3️⃣ 確認節點狀態

```bash
kubectl get nodes
```

此時通常會是 `NotReady`（尚未安裝 CNI）

---

## 五、安裝 CNI（Flannel）

```bash
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

等待 1~2 分鐘後：

```bash
kubectl get nodes
```

狀態應為：

```text
Ready
```

---

## 六、允許 control-plane 也能跑 Pod（學習環境）

```bash
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
```

---

## 七、測試部署第一個應用

```bash
kubectl create deployment nginx --image=nginx
kubectl expose deployment nginx --type=NodePort --port=80
kubectl get svc
```

使用瀏覽器或 curl 測試：

```bash
curl http://<NODE-IP>:<NODE-PORT>
```

---

## 八、結語

你現在已完成：

* Docker 安裝
* Kubernetes 正式安裝
* CNI 網路設定
* 第一個 Pod / Deployment

這是一個 **標準、符合 2025 官方規範** 的 Ubuntu 24.04 Kubernetes 教學流程，適合：

* 自學
* 寫技術文件
* 內部教育訓練

---

## 延伸學習建議

* Pod / Deployment / ReplicaSet 差異
* Service（ClusterIP / NodePort / LoadBalancer）
* Ingress
* Helm
* kind / minikube vs kubeadm


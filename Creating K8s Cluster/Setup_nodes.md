## Setting up a **Kubernetes cluster** with two **remote Debian-based nodes**

Setting up a **Kubernetes cluster** with two **remote Debian-based nodes** (one as a control plane and one as a worker) requires careful setup. Below is a **step-by-step guide** to deploy an efficient **Kubernetes cluster** using **kubeadm**, which is the recommended tool.

---

## **Prerequisites**

  - **Two remote Debian-based nodes** (One for Control Plane, One for Worker)  
  - **User with sudo privileges** on both nodes  
  - **Minimum hardware requirements**:
  - **Control Plane Node**: 2 CPU, 4GB RAM
  - **Worker Node**: 2 CPU, 2GB RAM  
  - **Static IP Addresses** assigned to each node  
  - **Network connectivity** between the nodes  
  - **Open required firewall ports** (covered in the steps below)  

---

## Step 1: Setup Hostnames & Update System

### **On the Control Plane Node** (`cp-node`)

```sh
sudo hostnamectl set-hostname cp-node
```

### **On the Worker Node** (`worker-node`)

```sh
sudo hostnamectl set-hostname worker-node
```

**Update and upgrade all packages** on both nodes:

```sh
sudo apt update && sudo apt upgrade -y
```

---

## **Step 2: Disable Swap & Modify Kernel Parameters**

Kubernetes requires **swap to be disabled**. Run the following on both nodes:

```sh
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab
```

Now, apply necessary **kernel parameters**:

```sh
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF

sudo sysctl --system
```

---
## **Step 3: Install the dependencies for adding repositories**

```sh
apt-get update
apt-get install -y software-properties-common curl
```

---
## **Step 4: Add the Kubernetes repository on each node**
```sh
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.32/deb/Release.key |
   sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.32/deb/ /" |
   sudo tee /etc/apt/sources.list.d/kubernetes.list
```

---
## **Step 5: Add the CRI-O repository on each node**
```sh
curl -fsSL https://pkgs.k8s.io/addons:/cri-o:/stable:/v1.32/deb/Release.key |
    sudo gpg --dearmor -o /etc/apt/keyrings/cri-o-apt-keyring.gpg

echo "deb [signed-by=/etc/apt/keyrings/cri-o-apt-keyring.gpg] https://pkgs.k8s.io/addons:/cri-o:/stable:/v1.32/deb/ /" |
    sudo tee /etc/apt/sources.list.d/cri-o.list
```
---
## **Step 6: Install Kubernetes Components (kubeadm, kubelet, kubectl)**

Run these commands on **both nodes**, and Mark the packages as held back to prevent automatic installation, upgrade, or removal:

```sh
# curl -fsSL https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo gpg --dearmor -o /usr/share/keyrings/kubernetes-archive-keyring.gpg

# echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

---
## **Step 7: Start CRI-O**
```sh
sudo systemctl start crio.service

```

---
## **Step 8: Initialize Kubernetes Cluster (Control Plane)**

On the **Control Plane Node**, run:

```sh
# sudo kubeadm init --pod-network-cidr=192.168.0.0/16
sudo kubeadm init 
```

Once completed, follow the displayed instructions:

**Setup kubectl for your user**:

```sh
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

<!-- **Enable the control plane to schedule pods** (Optional, if you plan to run workloads on it):

```sh
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
``` -->

---
## **Step 9: Deploy a Network Plugin**

You need a CNI plugin for inter-pod communication.

For **Calico**, run:

```sh
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
```

---

## **Step 10: Join Worker Node**

On the **worker node**, use the `kubeadm join` command shown in the output of `kubeadm init`.  

If you missed it, retrieve the token from the control plane:

```sh
kubeadm token create --print-join-command
```
or

```sh
kubeadm token create --print-join-command --ttl 0
```

Run the displayed command **on the worker node**, which looks like this:

```sh
sudo kubeadm join <CONTROL_PLANE_IP>:6443 --token <TOKEN> --discovery-token-ca-cert-hash sha256:<HASH>
```
or 
```sh
kubeadm join 172.31.47.143:6443 --token bqrhd2.aocpuq2c5ug81fmo --discovery-token-ca-cert-hash sha256:5b2ad604804e2edc5367b35314ef190af8389d6a509f99a79ea7b35463a6727c 
```

---

## **Step 11: Verify Cluster**

On the **Control Plane Node**, check if the worker node has joined:

```sh
kubectl get nodes
```

You should see something like as below:

```sh
NAME          STATUS   ROLES           AGE    VERSION
cp-node       Ready    control-plane   10m    v1.32.0
worker-node   Ready    <none>          2m     v1.32.0
```

---

## **Step 12: Deploy a Test Application**

Run a simple Nginx deployment to test:

```sh
kubectl create deployment nginx --image=nginx
kubectl expose deployment nginx --type=NodePort --port=80
```

Find the **NodePort**:

```sh
kubectl get svc nginx
```

Then, access it using:

```
http://<NODE_IP>:<NODE_PORT>
```

---

## **Final Notes**

1. **Cluster Setup Completed!**  
2. **Control Plane & Worker Node are running Kubernetes**  
3. **Pod networking is enabled with Calico**  
4. **You can deploy apps and expose them**  



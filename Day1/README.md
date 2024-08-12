# Kubeadm Installation Guide
---
Kubeadm allows us to customize and configure our clusters according to our needs and preferences. We can configure things like the network plugin, authentication and authorization mechanisms, and storage solutions.

## Pre-requisites

- Ubuntu **24.04 LTS** installed on all machines (1 master node and at least 1 worker node).
- Minimum instance required to create cluster = 2 (Master 1 and Worker 1)
- Root or **sudo** access to the machines.
- At least **2GB of RAM** per machine (4GB recommended for the master node).
- Make sure your all instance are in same Security group.
- Expose port **6443** in the Security group, so that worker nodes can join the cluster.

## AWS Setup

- Make sure your all instance are in same **Security group**.
- Expose port **6443** in the **Security group**, so that worker nodes can join the cluster.

---

## Step 1 : Create an instance for master and worker on AWS EC2

- Go to AWS console -> EC2 -> Launch Instance
- AMI : Ubuntu 24.04
- Name : master-node

---

## Step 2 : Update and Upgrade the System
Start by updating the package list and upgrading the installed packages to the latest versions.

```bash
sudo apt update && sudo apt upgrade -y
```

---

## Step 3 : Install Docker
Kubernetes uses Docker as its container runtime. Install Docker on all nodes (master and worker nodes).

```bash
sudo apt install -y docker.io
```
Verify that Docker is installed correctly.
```bash
docker --version
```

---

## Step 4: Install Kubernetes Components
Install the Kubernetes components: kubeadm, kubelet, and kubectl on all nodes.

Add the Kubernetes apt repository:

```bash
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```

Install the Kubernetes components:
```bash
sudo apt update
sudo apt install -y kubelet kubeadm kubectl
```
Hold the packages at their current version to prevent automatic upgrades:
```bash
sudo apt-mark hold kubelet kubeadm kubectl
```

---

## Step 5 : Disable Swap

Kubernetes requires swap to be disabled. Disable swap on all nodes.

```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
 ``` 

---

## Step 6 : Initialize the Master Node
On the master node, initialize the Kubernetes cluster with kubeadm.

```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
```
After initialization, you'll see a join command. Copy this command as it will be used to join worker nodes to the cluster.

```bash
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join <master-node-ip>:6443 --token kwngvy.wl6ck69yg62an0ll \
    --discovery-token-ca-cert-hash sha256:355e0aa517177aa11ad3fd8293babfc194b0a4771c588d3502cdbe1bf49535c0
```

Copy the last command starts with kubeadm join, we need to join worker nodes. This command need to run in worker nodes.

---

## Step 7 : Configure kubectl for the Master Node
Set up the kubeconfig file for the root user on the master node.

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Verify the cluster status.

```bash
kubectl get nodes
```

---

## Step 8: Install a Pod Network Add-on on Master node

Install a pod network so that your pods can communicate with each other. We'll use Flannel for this example.

```bash
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

Verify that all nodes are up and running.

```bash
kubectl get nodes
```

---

## Step 9 : Join Worker Nodes to the Cluster

On each worker node, use the join command obtained from the master node initialization step. The command will look something like this:

```bash
sudo kubeadm join <master-ip>:<master-port> --token <token> --discovery-token-ca-cert-hash sha256:<hash>
```

Verify that the nodes have joined the cluster. Execute following command in master node.

```bash
kubectl get nodes
```

```bash
NAME     STATUS     ROLES           AGE   VERSION
docker   Ready      control-plane   10m   v1.30.1
worker   NotReady   <none>          16s   v1.30.1
```

---

## Step 10: Deploy a Test Application

Deploy a simple Nginx application to verify that your Kubernetes cluster is working correctly.

```bash
kubectl create deployment nginx --image=nginx
kubectl expose deployment nginx --port=80 --type=NodePort
```

(Optional) Disable the firewall or add ports in firewall

In this step, we can disable the firewall for temporary or we can add the node port and Nginx ports in firewall

Get the NodePort assigned to the Nginx service.

```bash
kubectl get svc
```

```bash
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
kubernetes   ClusterIP   10.96.0.1       <none>        443/TCP        11m
nginx        NodePort    10.102.78.125   <none>        80:30711/TCP   7s
```

You should be able to access the Nginx application by visiting http://<node-ip>:<node-port> in your web browser.

Note: Replace <node-ip> with your worker node and you can get the <node-port>. In the previous command output, the node port is 30711.

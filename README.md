
# Kubernetes Cluster Setup Guide (EC2 Instances)

## Table of Contents
1. [EC2 Configuration and Security Group Setup](#ec2-configuration-and-security-group-setup)
2. [Key Kubernetes Definitions](#key-kubernetes-definitions)
3. [Kubernetes Installation Options](#kubernetes-installation-options)
4. [Container Runtime Explanation and Installation](#container-runtime-explanation-and-installation)
5. [Installing `kubeadm`, `kubelet`, and `kubectl`](#installing-kubeadm-kubelet-and-kubectl)
6. [Initializing the Control Plane](#initializing-the-control-plane)
7. [Pod Network and Other Network Types](#pod-network-and-other-network-types)
8. [Joining Worker Nodes](#joining-worker-nodes)
9. [Verifying Cluster Status](#verifying-cluster-status)
10. [Additional Information](#additional-information)

---

## 1. EC2 Configuration and Security Group Setup

For this Kubernetes cluster, we are using three EC2 instances (one control plane node and two worker nodes). Below are the minimum configurations for each instance, along with the necessary security group settings.

### Minimum EC2 Instance Configuration

- **Control Plane Node**:
  - Instance type: `t2.medium` (minimum)
  - RAM: 4 GB
  - Storage: 20 GB (General-purpose SSD)
  - OS: Ubuntu 20.04
  - Elastic IP: Yes (for external access)

- **Worker Nodes**:
  - Instance type: `t2.micro` (minimum)
  - RAM: 1 GB
  - Storage: 20 GB (General-purpose SSD)
  - OS: Ubuntu 20.04
  - Elastic IP: Optional (recommended for external access)

### Security Group Setup

Ensure the following inbound rules are configured for the security group associated with all EC2 instances:

1. **SSH**: Port `22` (Source: `0.0.0.0/0` for public access or limit to your IP range).
2. **Kubernetes API Server**: Port `6443` (Source: Worker node IP ranges).
3. **NodePort Services**: Ports `30000-32767` (Source: `0.0.0.0/0` to access exposed services).
4. **Kubelet API**: Port `10250` (Source: Control Plane IP).
5. **ETCD**: Port `2379-2380` (Source: Control Plane IP for internal Kubernetes etcd communication).

---

## 2. Key Kubernetes Definitions

- **Node**: A machine in the Kubernetes cluster, either a control plane or worker node.
- **Control Plane**: Manages the overall cluster, handling scheduling, scaling, and managing worker nodes.
- **Pod**: The smallest deployable unit in Kubernetes, which represents one or more containers.
- **Worker Node**: A node that runs the actual workloads (containers).
- **Service**: An abstraction that defines a logical set of Pods and a policy for accessing them.
- **Namespace**: A virtual cluster that provides logical isolation for resources within Kubernetes.
- **Pod Network**: A network that enables communication between Kubernetes Pods.

---

## 3. Kubernetes Installation Options

Kubernetes can be set up in various ways depending on your environment:

1. **Minikube**: Suitable for local development, it runs a single-node Kubernetes cluster on your local machine.
2. **kubeadm**: Ideal for production-level clusters, kubeadm helps initialize the control plane and join worker nodes.
3. **Amazon EKS**: A managed Kubernetes service on AWS, ideal for large-scale production deployments.

In this guide, we will use **kubeadm** for setting up a multi-node Kubernetes cluster on EC2 instances.

---

## 4. Container Runtime Explanation and Installation

A **container runtime** is responsible for running containers. Kubernetes interacts with different container runtimes via the Container Runtime Interface (CRI).

### Common Container Runtimes:
1. **containerd**: Lightweight and widely used with Kubernetes.
2. **CRI-O**: Optimized for Kubernetes and supports OCI-compliant containers.
3. **Docker**: Kubernetes uses `cri-dockerd` to interface with Docker.

For this guide, we will use `containerd` as the runtime. However, you can choose any other runtime.

### Containerd Installation

1. **Create a script for installing `containerd`**:
   ```bash
   vim containerd-install.sh
   ```

2. **Script contents**:
   ```bash
   #!/bin/bash
   sudo apt-get update
   sudo apt-get install -y containerd
   sudo mkdir -p /etc/containerd
   sudo containerd config default | sudo tee /etc/containerd/config.toml
   sudo systemctl restart containerd
   sudo systemctl enable containerd
   ```

3. **Make the script executable and run it**:
   ```bash
   chmod +x containerd-install.sh
   ./containerd-install.sh
   ```

4. **Verify installation**:
   ```bash
   sudo systemctl status containerd
   ```

---

## 5. Installing `kubeadm`, `kubelet`, and `kubectl`

After installing the container runtime, install the Kubernetes components:

1. **Create the installation script**:
   ```bash
   vim k8s-install.sh
   ```

2. **Script contents**:
   ```bash
   #!/bin/bash
   sudo apt-get update
   sudo apt-get install -y apt-transport-https ca-certificates curl
   curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
   echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
   sudo apt-get update
   sudo apt-get install -y kubelet kubeadm kubectl
   sudo apt-mark hold kubelet kubeadm kubectl
   ```

3. **Make the script executable and run it**:
   ```bash
   chmod +x k8s-install.sh
   ./k8s-install.sh
   ```

4. **Verify installation**:
   ```bash
   kubeadm version
   ```

---

## 6. Initializing the Control Plane

To initialize the control plane, run the following command on the **control plane** node:

```bash
sudo kubeadm init
```

After the control plane is initialized, set up your kubeconfig file to use `kubectl`:

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### Verify the Control Plane:

```bash
kubectl get nodes
```

---

## 7. Pod Network and Other Network Types

Kubernetes uses a **Pod Network** to allow Pods to communicate across nodes. Without a network plugin, Pods will not be able to communicate across the cluster.

### Types of Kubernetes Networks

1. **Pod Network (CNI)**: This allows communication between Kubernetes Pods. Examples include:
   - **Calico**: Provides network security policies and is widely used.
   - **Flannel**: A simple overlay network.
   - **Weave**: Provides both networking and network policy enforcement.

2. **Service Network**: Exposes Kubernetes services to allow access from other pods or external traffic.
3. **Intra-Node and Inter-Node Networking**: Handles communication within the same node and between nodes.

### Installing a Pod Network (Calico)

Install **Calico** for the pod network:

```bash
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
```

Verify the pod network:

```bash
kubectl get pods -n kube-system
```

---

## 8. Joining Worker Nodes

To join the worker nodes to the cluster, run the `kubeadm join` command provided by the control plane during initialization on each worker node.

Example:

```bash
kubeadm join <control-plane-ip>:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>
```

### Verify Worker Node Status:

On the control plane node, check the status of all nodes:

```bash
kubectl get nodes
```

---

## 9. Verifying Cluster Status

Once all nodes have joined the cluster, verify the overall cluster status by running:

```bash
kubectl get nodes
kubectl get pods --all-namespaces
```

Check that all nodes are in the `Ready` state and that all pods are running without issues.

---

## 10. Additional Information

Kubernetes provides additional features like:
- **Horizontal Pod Autoscaling**: Automatically scale Pods based on resource usage.
- **Ingress Controllers**: Manage external access to services.
- **Load Balancing**: Balance traffic across multiple Pods.

Feel free to explore these advanced features as your Kubernetes expertise grows.



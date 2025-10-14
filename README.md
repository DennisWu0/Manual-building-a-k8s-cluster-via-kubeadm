# Manual-building-a-k8s-cluster-via-kubeadm

🧩 This guide walks you through a complete, manual Kubernetes cluster setup using **kubeadm** and **containerd**, without relying on managed services or automation tools.

I write this article cause I’ve seen many resources online, but there are no resources that explain Kubernetes in a clear, step-by-step way. That’s why I wrote this guide to help developers, DevOps engineers, and system administrators understand how Kubernetes really works under the hood, from system setup to networking and CNI integration.

## 📘 Overview

In this guide, we’ll manually build a **multi-node Kubernetes cluster** using:

- **1 Master (Control Plane)**
- **2 Worker Nodes**

All machines will communicate via a **bridged network**, and **Calico** will serve as the **CNI (Container Network Interface)** plugin.

# **Prerequisites:**

In this guide, we’ll use **VMware Workstation Pro** to create our virtual machines that will act as **K8s** **servers**.

Each virtual machine will represent one of the following roles:

- **1 Master Node** – the control plane that manages the cluster
- **2 Worker Nodes** – where containers (Pods) will actually run

We’ll configure all three VMs to be on the **same bridged network**, so they can communicate with each other as if they were physical machines on the same LAN.

We can choose any lightweight **Linux distribution** (for example, Ubuntu Server 22.04 LTS).

### **The minimum requirements:**

| Resource | Minimum Requirement |
| --- | --- |
| **Memory** | ≥ 2 GB RAM per machine |
| **CPU** | ≥ 2 vCPUs on the master node |
| **Disk Space** | ≥ 20 GB |
| **Network** | Full connectivity between nodes (bridged) |

<br>
#NOTE: The following outlines the process for securing servers. If you have already completed this step, you can skip it and proceed to **The main steps for building the Kubernetes environment**.
<br>

### SSH & Networking Configuration

**SSH Setup:** 

By default, SSH runs on port 22, which is commonly scanned by bots. To improve security, change SSH port to something in a high range (for example, 50000) and then allow access through firewall from local machine. ****This practice I did in each vm environment helps protect servers from common automated SSH attacks.

```bash
petops@master:~$ sudo systemctl status ssh
[sudo] password for petops:
● ssh.service - OpenBSD Secure Shell server
     Loaded: loaded (/usr/lib/systemd/system/ssh.service; enabled; preset: enabled)
     Active: active (running) since Sun 2025-10-12 06:11:05 UTC; 23h ago
 Invocation: 9d1a5cc357af47ba8dfce770afa47cb9
TriggeredBy: ● ssh.socket
       Docs: man:sshd(8)
             man:sshd_config(5)
   Main PID: 13816 (sshd)
      Tasks: 1 (limit: 3922)
     Memory: 3.4M (peak: 19.7M)
        CPU: 215ms
     CGroup: /system.slice/ssh.service
             └─13816 "sshd: /usr/sbin/sshd -D [listener] 0 of 10-100 startups"

Oct 12 06:11:05 master sshd[13816]: Server listening on 0.0.0.0 port 50001.
Oct 12 06:11:05 master systemd[1]: Started ssh.service - OpenBSD Secure Shell server.
Oct 12 06:11:05 master sshd[13816]: Server listening on :: port 50001.
Oct 12 16:50:59 master sshd-session[31560]: Accepted password for petops from 192.168.0.223 port 59266 ssh2
Oct 12 16:50:59 master sshd-session[31560]: pam_unix(sshd:session): session opened for user petops(uid=1001) by p>
```

**Configure Static IP via Netplan:**
We’ll assign a **static IP** to each server so that Kubernetes components can reach each other consistently.

Example Netplan configuration (`/etc/netplan/00-installer-config.yaml`):

```bash
network:
  version: 2
  ethernets:
    ens33:
      dhcp4: false
      addresses:
        - 192.168.0.201/24
      routes:
        - to: default
          via: 192.168.0.1
      nameservers:
        addresses: [8.8.8.8,8.8.4.4]
```

Apply changes:

```bash
sudo netplan apply

```

**Set Hostname and Hosts File**

Assign a hostname to each node:

```bash
hostnamectl set-hostname [your-master-server-name]
```

Then, map IPs to hostnames in `/etc/hosts` so nodes can resolve each other by name:

```bash
petops@master:~$ cat /etc/host
192.168.0.201 master
192.168.0.202 worker1
192.168.0.203 worker2
```

**Configure the whitelist via ufw:** 

As a security measure, I added the same network to the whitelist

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow from 192.168.0.0/24
```

💡 You can restrict specific nodes with:

```
sudo ufw allow from 192.168.0.223 to any port 6443 proto tcp
```

Enable the firewall:

```bash
sudo ufw enable
```

Check its status:

```bash
sudo ufw status verbose
```

# The main step for building K8s environment

### Step 1: Create a Dedicated User

We should never use the `root` user for regular K8s management. So instead, using a `non-root` user for improving operational security. This is a best Practice to avoid using root for administrative K8s tasks.

```bash
 adduser petops
 usermod -aG sudo petops
 newgrp petops
```

### Step 2: Disable Swap

K8s requires **swap to be disabled**. If swap is on, the kubelet service will refuse to start.

```bash
sudo swapoff -a
```

Then, permanently comment out the swap line in `/etc/fstab`:

```bash
sudo sed -i '/swap/ s/^/#/' /etc/fstab
```

Expected result in `/etc/fstab`:

```bash
#/swap.img      none    swap    sw      0       0
```

### Step 3: Enable Kernel Modules for Networking

Before K8s can manage networking, the OS must allow **packet forwarding** and **bridge filtering**.

```bash
sudo modprobe overlay
sudo modprobe br_netfilter
cat <<EOF | sudo tee /etc/sysctl.d/kubernetes.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system
```

-> This step opens up the Linux system to allow K8s networking to work. It lets pods talk to each other, and lets kube-proxy control traffic safely.

### Step 4: Install Container Runtime

There are 3 container runtimes: containerd, docker, and cri

Historically, K8s used Docker as its container runtime.
But since Kubernetes v1.24, Docker support was removed and replaced by containerd (and CRI-O). So we use containerd in this case instead of Docker or cri

**Add repo and install packages**

```bash
sudo apt install -y curl gnupg2 software-properties-common apt-transport-https ca-certificates
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmour -o /etc/apt/trusted.gpg.d/docker.gpg
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"

```

**Install containerd**

```bash
sudo apt update -y
sudo apt install -y [containerd.io](http://containerd.io/)
```

**Config containerd**

By default, containerd doesn’t use systemd for cgroup management, but K8s expects it.

We’ll fix that by editing the configuration file.

```bash
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml

sudo systemctl restart containerd
sudo systemctl enable containerd
```

### Step 5: Install K8s Components

Some users encounter this key error, that is because the K8s repository now uses a new keyring format.

```bash
W: OpenPGP signature verification failed: https://prod-cdn.packages.k8s.io/repositories/isv:/kubernetes:/core:/stable:/v1.30/deb  InRelease: The following signatures couldn't be verified because the public key is not available: NO_PUBKEY 234654DA9A296436
E: The repository 'https://pkgs.k8s.io/core:/stable:/v1.30/deb  InRelease' is not signed.
N: Updating from such a repository can't be done securely, and is therefore disabled by default.
N: See apt-secure(8) manpage for repository creation and user configuration details.
N: Some sources can be modernized. Run 'apt modernize-sources' to do so.
```

Previously, the keyring file type was unsupported. So I have to fix it by correctly adding the K8s signing key using `Release.key` and storing it as a `.gpg` file under `/etc/apt/keyrings`.

```bash
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | \
  sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-archive-keyring.gpg

echo "deb [signed-by=/etc/apt/keyrings/kubernetes-archive-keyring.gpg] \
https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /" | \
sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update -y
sudo apt-get install -y kubelet kubeadm kubectl
```

Holding the recent version of kubelet kubeadm kubectl cause when we use the update syntax, some components will be updated, leading to our k8s cluster getting errors

```bash
sudo apt-mark hold kubelet kubeadm kubectl
```

### Step 6: Initialize the Cluster (Control Plane)

On the **master node**, run:

```bash
sudo kubeadm init
```

K8s will tell you where the syntax will be used:

```bash
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

kubeadm join 192.168.0.201:6443 --token gpthzo.a03lztlp91fxon5n \
        --discovery-token-ca-cert-hash sha256:c59fc8bae0f60edf2e02cb60f2028c0ed9542320916c6395fa4a22cf7ec89eaa
```

Use these in master node 

```bash
  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

And copy these syntaxs to worker nodes to tell k8s that these vms are worker nodes:

```bash
sudo kubeadm join 192.168.0.201:6443 --token gpthzo.a03lztlp91fxon5n \
        --discovery-token-ca-cert-hash sha256:c59fc8bae0f60edf2e02cb60f2028c0ed9542320916c6395fa4a22cf7ec89eaa
```

### Step 7: Install a CNI

After joining, nodes might show as **NotReady**:

```bash
petops@master:~$ kubectl get nodes
NAME      STATUS     ROLES           AGE     VERSION
master    NotReady   control-plane   6m51s   v1.30.14
worker1   NotReady   <none>          44s     v1.30.14
worker2   NotReady   <none>          37s     v1.30.14
```

When we check the `kubectl describe nodes worker1` , this error means the **network plugin (CNI)** is missing.

```bash
Conditions:
  Type             Status  LastHeartbeatTime                 LastTransitionTime                Reason                       Message
  ----             ------  -----------------                 ------------------                ------                       -------
  MemoryPressure   False   Sun, 12 Oct 2025 17:40:10 +0000   Sun, 12 Oct 2025 17:39:39 +0000   KubeletHasSufficientMemory   kubelet has sufficient memory available
  DiskPressure     False   Sun, 12 Oct 2025 17:40:10 +0000   Sun, 12 Oct 2025 17:39:39 +0000   KubeletHasNoDiskPressure     kubelet has no disk pressure
  PIDPressure      False   Sun, 12 Oct 2025 17:40:10 +0000   Sun, 12 Oct 2025 17:39:39 +0000   KubeletHasSufficientPID      kubelet has sufficient PID available
  Ready            False   Sun, 12 Oct 2025 17:40:10 +0000   Sun, 12 Oct 2025 17:39:39 +0000   KubeletNotReady              container runtime network not ready: NetworkReady=false reason:NetworkPluginNotReady message:Network plugin returns error: cni plugin not initialized
```

**Installing CNI-Calico for k8s cluster**

```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.30.3/manifests/calico.yaml
```

Wait a few minutes, then check again:

```bash
petops@master:~$ kubectl get nodes
NAME      STATUS   ROLES           AGE   VERSION
master    Ready    control-plane   18m   v1.30.14
worker1   Ready    <none>          12m   v1.30.14
worker2   Ready    <none>          12m   v1.30.14
```

🎉 Congratulations! You now have a fully functional K8s cluster built manually from scratch.

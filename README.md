### Kubernetes Setup Guide for Virtual Machines

Welcome to the Kubernetes Setup Guide! This repository provides a streamlined, step-by-step approach for setting up a Kubernetes cluster on virtual machines. Ideal for personal use or for sharing with others, this guide simplifies the process of getting a Kubernetes environment up and running.

#### What’s Included:

-   **Pre-requisites:** Lists software, tools, and resources you'll need.
-   **VM Configuration:** Steps for setting up and configuring virtual machines.
-   **Kubernetes Installation:** Clear instructions for installing and setting up Kubernetes on your VMs.
-   **Cluster Setup:** How to initialize and configure your Kubernetes cluster.
-   **Testing and Validation:** Methods to test clusters and ensure everything is working correctly.
-   **Troubleshooting:** Common issues and their solutions.

#### Quick Start:


1.  **Follow the Guide:** Execute the step-by-step instructions for setting up your VMs and Kubernetes.
2.  **Test Your Setup:** Deploy sample applications to verify your cluster is working as expected.

This guide is crafted for my personal use but is freely available for others to follow and fork. It provides a step-by-step approach to setting up a Kubernetes cluster on virtual machines, including VM configuration, Kubernetes installation, cluster setup, and testing. Feel free to use, modify, and contribute to this repository!

<hr>

### Prerequisites

Before you start setting up Kubernetes on virtual machines, make sure you have the following tools and configurations in place:

#### 1. Visual Studio Code (VS Code)

- Download and Install: [Visual Studio Code](https://code.visualstudio.com/)
-   Required Extensions:
    1.  Remote - SSH
        -   Description: Connect to remote machines over SSH.
        -   URL: [Remote - SSH by Microsoft](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-ssh)
        -   Extension ID: `ms-vscode-remote.remote-ssh`
    2.  Remote - SSH: Editing Configuration Files
        -   Description: Easily edit SSH configuration files.
        -   URL: [Remote - SSH: Editing Configuration Files](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-ssh-edit)
        -   Extension ID: `ms-vscode-remote.remote-ssh-edit`
    3.  Remote Explorer
        -   Description: View and manage remote connections.
        -   URL: [Remote Explorer](https://marketplace.visualstudio.com/items?itemName=ms-vscode.remote-explorer)
        -   Extension ID: `ms-vscode.remote-explorer`

**Note:** These tools and extensions are optional. They are recommended for easing file system navigation and terminal usage, as they provide a more user-friendly interface compared to the default terminal on servers, which can be restrictive and less convenient for managing files for beginners.

#### 2. Virtual Machine Platform

-   Required Platform: VMware or any compatible virtual machine platform.
-   VM Configuration:
    -   Number of VMs: 2, 1 Master node and 1 worker node
    -   Operating System: Linux Ubuntu Server
    -   Settings: Use default settings for Ubuntu installation.
    -   Software Installation: Ensure `openssh` and `net-tools` are installed.
    -   Avoid Installing: **Do not install** default **Docker** or **Kubernetes** packages to prevent conflicts with the setup process.

Ensure these tools and configurations are in place before proceeding with the Kubernetes setup.

<hr>

### Connecting to SSH Using Visual Studio Code

To make the process of connecting to a remote machine easier, we will use the Visual Studio Code GUI method. This approach is user-friendly and simplifies SSH configuration without needing to use the terminal.

1.  Open Remote Explorer:
    
    -   Click on the "Remote Explorer" icon located on the sidebar of Visual Studio Code.

2.  Select the SSH Option:
    
    -   In the "Remote Explorer" panel, ensure that the "Remotes (Tunnels/SSH)" dropdown is selected.

3.  Add a New Remote:
    -   Click the **"+" (Add New Remote)** button in SSH (as shown in the image above).
    -   This action will open a prompt where you can enter the details of your remote connection, such as the SSH address of your host.

4.  Enter SSH Details:
	- Enter the SSH details in the format `username@host-or-ip` . For example, `user123@192.168.1.10` .
	- You may be prompted to specify the port number (default is `22` for SSH) and to save the SSH key or password for easier access in the future.

5.  Connect to the Remote Host:
    
    -   Once the connection details are saved, you can easily connect to your remote host by selecting it from the list of available remotes in the "Remote Explorer".
6.  Verify Connection:
    
    -   A new window should open, indicating a successful connection to your remote server.

By following these steps, you can easily manage your remote connections using the Visual Studio Code GUI, making it simpler to work with remote machines for Kubernetes setup.

<hr>

### VM Configuration 

**Note:** Please repeat the step 1-5 for both virtual machines. One for master node and another for worker node.

#### Step 1: Disable swap

Swap space on a hard drive is used by operating systems to extend RAM capacity by moving less frequently accessed data to the disk. However, because accessing data on a hard drive is much slower than accessing data in RAM, using swap can significantly reduce performance.

Kubernetes relies on accurate resource information to schedule workloads efficiently. When swap is used, it can interfere with Kubernetes' ability to assess available resources correctly, leading to potential scheduling issues.

To avoid this, it's recommended to disable swap before installing Kubernetes. You can temporarily disable swap with:
```bash 
sudo swapoff -a
```

To keep it disabled after a reboot, modify the configuration file with:
```bash 
sudo sed -i '/ swap / s/^/#/' /etc/fstab
```

####  Step 2: Set up hostnames

Hostnames are human-readable names assigned to machines for identification within a network. In a Kubernetes cluster, each node must have a unique hostname so that Kubernetes can distinguish between them.

To set a hostname for your master node, run the following command on the chosen machine or VM instance:
```bash 
sudo hostnamectl set-hostname "master-node"
```

Then, refresh the bash session to apply the new hostname immediately by running:
```bash 
exec bash
```

This will ensure that the master node is correctly identified in the cluster.

Repeat the hostname-setting commands on all other nodes in the Kubernetes cluster, adjusting the hostname for each. For example, on a worker node, use:
```bash 
sudo hostnamectl set-hostname "worker-node1"
```

####  Step 3: Set up the IPV4 bridge on all nodes
To configure the IPV4 bridge on all nodes, execute the following commands on each node.

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# sysctl params required by setup, params persist across reboots
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# Apply sysctl params without reboot
sudo sysctl --system
```

#### Step 4: Install `kubelet`, `kubeadm`, and `kubectl` on Each Node

To set up your Kubernetes cluster, you need to install `kubelet`, `kubeadm`, and `kubectl` on every node.

-   **Kubelet**: This agent runs on all nodes, ensuring that containers specified in Pods are running correctly. (Pods are the smallest units that can be deployed in a Kubernetes cluster.)
-   **Kubeadm**: This tool is used to initialize and set up the Kubernetes cluster, including configuring the master node and enabling worker nodes to join.
-   **Kubectl**: A command-line tool to manage Kubernetes clusters, allowing you to deploy applications, inspect resources, and perform cluster management tasks from the terminal.

Before installing these tools, make sure to update the package index by running:
```bash
# apt-transport-https may be a dummy package; if so, you can skip that package
# If the directory `/etc/apt/keyrings` does not exist, it should be created before the curl command, read the note below.
# sudo mkdir -p -m 755 /etc/apt/keyrings
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gpg

#Download the public signing key for the Kubernetes package repositories. The same signing key is used for all repositories so you can disregard the version in the URL
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```

Add the Kubernetes APT repository to your system. Note that this repository provides packages only for Kubernetes version 1.31. If you want to install a different minor version, modify the version number in the repository URL accordingly. Make sure to refer to the documentation that matches the version you plan to install.
```bash
# This overwrites any existing configuration in /etc/apt/sources.list.d/kubernetes.list
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

Update the `apt` package index, install kubelet, kubeadm and kubectl, and pin their version.
```bash
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
sudo systemctl enable --now kubelet
```

#### Step 5: Install Docker
Docker is a platform that allows you to create, distribute, and run applications in containers, which offer a lightweight and consistent environment across different systems. In Kubernetes, Docker acts as the container runtime, playing a critical role in deploying and managing containerized applications.

Containerd is a lightweight, industry-standard container runtime that handles essential tasks like starting, stopping, and managing containers. It is designed for simplicity and high performance, ensuring Kubernetes can efficiently manage containers while maintaining compatibility across environments.

To install Docker and containerd, use the following command:
```bash 
sudo apt install docker.io containerd
```

Next, configure containerd on all nodes to ensure its compatibility with Kubernetes. First, create a folder for the configuration file with the following command:
```bash 
sudo mkdir /etc/containerd
``` 

Then, create a default configuration file for containerd and save it as config.toml using the following command:
```bash 
sudo sh -c "containerd config default > /etc/containerd/config.toml"
```

After running these commands, you need to modify the config.toml file to locate the entry that sets "SystemdCgroup" to false and changes its value to true. This is important because Kubernetes requires all its components, and the container runtime uses [systemd](https://www.cherryservers.com/blog/an-ultimate-guide-of-how-to-manage-linux-systemd-services-with-systemctl-command) for cgroups.
``` bash 
sudo sed -i 's/ SystemdCgroup = false/ SystemdCgroup = true/' /etc/containerd/config.toml
```

Next, restart containerd and kubelet services to apply the changes you made on all nodes.
```bash
sudo systemctl restart containerd.service
sudo systemctl restart kubelet.service
sudo systemctl enable kubelet.service
```

> Until here will be same for both master and worker node. After this we mainly work on master node.

#### Step 6: Initialize the Kubernetes cluster on the master node

When setting up a Kubernetes control plane with kubeadm, it deploys various components like `kube-apiserver`, `kube-controller-manager`, `kube-scheduler`, `etcd`, and `kube-proxy` to handle and organize the cluster. To get these components, you need to download their images using a specific command.

```bash
sudo kubeadm config images pull
```

Next, initialize your master node. The `--pod-network` flag is setting the IP address range for the pod network. It will dynamically assign IP address.
```bash
sudo kubeadm init --pod-network
```
To manage the cluster, configure `kubectl` on the master node by creating a `.kube` directory in your home folder and copying the cluster's admin configuration there. After copying, update the file's ownership to grant the user permission to use it for cluster interactions. Here are the commands for these steps.

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

#### Step 7: Configure kubectl and Calico
Run the following commands on the master node to deploy the Calico operator.
```bash
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/tigera-operator.yaml
```

> WIP --todo make sure steps after this needed or not. add configure docker port part.

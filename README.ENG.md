# Single-Node Deployment

### This guide walks you through:
1. Setting up a single-node Kubernetes cluster using RKE.
2. Installing the Kubernetes Dashboard and application services.
3. Deploying the Envoy Gateway to enable external access to these services over the Internet.

### Prerequisites
An Ubuntu server (minimum version: 24.04.1) with:
- An external IP address.
- At least 32 GB of RAM.

### Instructions

1. Update the System

Update the package lists and upgrade installed packages:

```bash
sudo apt-get update
sudo apt-get upgrade -y
sudo apt-get dist-upgrade -y
```

2. Install Required Tools

Install net-tools for network configuration:

```bash
sudo apt-get install -y net-tools
```

3. Configure the Time Zone

Change the default time zone to one appropriate for your region:

```bash
sudo timedatectl set-timezone Europe/Moscow
```

4. Configure apt for HTTPS Repositories

Install the necessary tools for downloading repositories over HTTPS:

```bash
sudo apt install -y curl apt-transport-https ca-certificates software-properties-common
```

5. Install Docker

Add the Docker GPG key:

```bash
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
```

Add the Docker repository:

```bash
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

Update the package list and install Docker:

```bash
sudo apt update
sudo apt install -y docker-ce
```

Add your user to the Docker group:

```bash
sudo groupadd docker
sudo usermod -aG docker $USER
```

6. Install Helm

Add the Helm GPG key:

```bash
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
```

Add the Helm repository:

```bash
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
```

Update the package list and install Helm:

```bash
sudo apt update
sudo apt install -y helm
```

7. Install kubectl

Add the Kubernetes GPG key:

```bash
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```

Add the Kubernetes repository:

```bash
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

Update the package list and install kubectl:

```bash
sudo apt update
sudo apt install -y kubectl
```

8. Install the rke Command-Line Tool

Download the RKE binary:

```bash
wget https://github.com/rancher/rke/releases/download/v1.6.3/rke_linux-amd64 -O /usr/local/bin/rke
```

Make it executable:

```bash
sudo chmod +x /usr/local/bin/rke
```

9. Generate Certificates

Generate a self-signed certificate for your Kubernetes cluster:

```bash
openssl req -newkey rsa:4096 -x509 -sha256 -days 3650 -nodes -out cluster.crt -keyout cluster.key
```

10. Generate an SSH Key

Create an SSH key for accessing the node:

```bash
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519 -P ""
```

Add the key to the ~/.ssh/authorized_keys file for the user that will connect to the node during deployment.

11. Configure the Cluster

Run the rke command to generate a configuration file:

```bash
rke config
```

This command will guide you through the process of creating a configuration file for your cluster.

Set the KUBECONFIG environment variable to the path of the configuration file:

```bash
export KUBECONFIG=$(pwd)/kube_config_cluster.yml
```

Launch the cluster:
```bash
rke up
```
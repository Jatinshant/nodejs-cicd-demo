# CI/CD Pipeline with GitHub, Jenkins, DockerHub and Kubernetes



## Project Overview



This project demonstrates a complete CI/CD pipeline using GitHub, Jenkins, DockerHub, and Kubernetes.



Whenever code is pushed to GitHub, Jenkins automatically triggers a pipeline using GitHub Webhook. Jenkins pulls the latest code, builds a Docker image, pushes the image to DockerHub, and deploys the application to a Kubernetes cluster.



---



## Architecture



```text

Developer

   |

   v

GitHub Repository

   |

   v

GitHub Webhook

   |

   v

Jenkins Server

   |

   |-- Checkout Code

   |-- Build Docker Image

   |-- Push Image to DockerHub

   |-- Deploy to Kubernetes

   |

   v

Kubernetes Cluster

   |

   |-- Master Node

   |-- Worker Node

   |

   v

Application Running on NodePort

```



---



## Resources Used



| Resource   | Purpose                            |

| ---------- | ---------------------------------- |

| AWS EC2    | Servers for Jenkins and Kubernetes |

| GitHub     | Source code repository             |

| Jenkins    | CI/CD automation tool              |

| Docker     | Build container image              |

| DockerHub  | Store Docker images                |

| Kubernetes | Run containerized application      |

| kubeadm    | Create Kubernetes cluster          |

| containerd | Container runtime for Kubernetes   |

| Calico     | Kubernetes networking              |



---



## EC2 Instances



Three EC2 instances were launched.



| Instance Name  | Type     | OS           | Purpose                       |

| -------------- | -------- | ------------ | ----------------------------- |

| jenkins-server | t3.small | Ubuntu 24.04 | Jenkins, Docker, Git, kubectl |

| k8s-master     | t3.small | Ubuntu 24.04 | Kubernetes control plane      |

| k8s-worker     | t3.small | Ubuntu 24.04 | Runs application pods         |



Recommended storage:



```text

20 GB gp3 volume for each instance

```



---



## Security Groups



### Jenkins Security Group



Allow inbound rules:



```text

22      SSH        My IP

8080    Jenkins    0.0.0.0/0

```



Port `8080` is opened publicly because GitHub Webhook needs to reach Jenkins.



---



### Kubernetes Master Security Group



Allow inbound rules:



```text

22      SSH              My IP

6443    Kubernetes API   Jenkins SG / Worker SG / My IP

10250   Kubelet          Worker SG / Jenkins SG

```



---



### Kubernetes Worker Security Group



Allow inbound rules:



```text

22             SSH              My IP

10250          Kubelet          Master SG

30000-32767    NodePort Range   0.0.0.0/0

```



For this project, the application is exposed on:



```text

30080

```



---



# 1. Jenkins Server Setup



SSH into Jenkins server:



```bash

ssh -i your-key.pem ubuntu@<JENKINS_PUBLIC_IP>

```



Update server:



```bash

sudo apt update

sudo apt upgrade -y

```



---



## Install Java



Jenkins requires Java.



```bash

sudo apt install openjdk-17-jdk -y

```



Verify:



```bash

java -version

```



---



## Install Git



```bash

sudo apt install git -y

```



Verify:



```bash

git --version

```



---



## Install Docker



```bash

curl -fsSL https://get.docker.com | sudo sh

```



Add Ubuntu user to Docker group:



```bash

sudo usermod -aG docker ubuntu

```



Later after Jenkins installation, add Jenkins user also:



```bash

sudo usermod -aG docker jenkins

```



Restart Docker:



```bash

sudo systemctl restart docker

```



Verify:



```bash

docker --version

docker ps

```



---



## Install Jenkins



Add Jenkins key:



```bash

sudo mkdir -p /usr/share/keyrings

curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key | sudo tee /usr/share/keyrings/jenkins-keyring.asc > /dev/null

```



Add Jenkins repository:



```bash

echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/ | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null

```



Install Jenkins:



```bash

sudo apt update

sudo apt install jenkins -y

```



Start Jenkins:



```bash

sudo systemctl enable jenkins

sudo systemctl start jenkins

```



Check Jenkins status:



```bash

sudo systemctl status jenkins

```



Get Jenkins initial password:



```bash

sudo cat /var/lib/jenkins/secrets/initialAdminPassword

```



Open Jenkins:



```text

http://<JENKINS_PUBLIC_IP>:8080

```



---



## Install kubectl on Jenkins Server



Jenkins needs `kubectl` to deploy the application to Kubernetes.



```bash

curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"

```



```bash

chmod +x kubectl

sudo mv kubectl /usr/local/bin/

```



Verify:



```bash

kubectl version --client

```



---



## Install AWS CLI



```bash

curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o awscliv2.zip

```



```bash

sudo apt install unzip -y

unzip awscliv2.zip

sudo ./aws/install

```



Verify:



```bash

aws --version

```



---



## Fix Jenkins Docker Permission



Create Jenkins home directory if required:



```bash

sudo mkdir -p /home/jenkins

sudo chown -R jenkins:jenkins /home/jenkins

```



Add Jenkins user to Docker group:



```bash

sudo usermod -aG docker jenkins

```



Restart services:



```bash

sudo systemctl restart docker

sudo systemctl restart jenkins

```



Verify Docker access as Jenkins user:



```bash

sudo -u jenkins docker ps

```



---



# 2. Kubernetes Master Setup



SSH into master server:



```bash

ssh -i your-key.pem ubuntu@<K8S_MASTER_PUBLIC_IP>

```



Update server:



```bash

sudo apt update

sudo apt upgrade -y

```



---



## Disable Swap



Kubernetes requires swap to be disabled.



```bash

sudo swapoff -a

```



Edit fstab:



```bash

sudo nano /etc/fstab

```



Comment any swap line by adding `#` at the start.



Verify:



```bash

free -h

```



---



## Load Kernel Modules



```bash

cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf

overlay

br_netfilter

EOF

```



```bash

sudo modprobe overlay

sudo modprobe br_netfilter

```



---



## Configure Kubernetes Networking



```bash

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf

net.bridge.bridge-nf-call-iptables = 1

net.bridge.bridge-nf-call-ip6tables = 1

net.ipv4.ip_forward = 1

EOF

```



Apply changes:



```bash

sudo sysctl --system

```



---



## Install containerd



```bash

sudo apt install containerd -y

```



Create containerd config:



```bash

sudo mkdir -p /etc/containerd

containerd config default | sudo tee /etc/containerd/config.toml

```



Edit config:



```bash

sudo nano /etc/containerd/config.toml

```



Find:



```text

SystemdCgroup = false

```



Change it to:



```text

SystemdCgroup = true

```



Restart containerd:



```bash

sudo systemctl restart containerd

sudo systemctl enable containerd

```



---



## Install kubeadm, kubelet and kubectl



```bash

sudo apt-get install -y apt-transport-https ca-certificates curl gpg

```



```bash

sudo mkdir -p /etc/apt/keyrings

```



```bash

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

```



```bash

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

```



```bash

sudo apt update

sudo apt install kubelet kubeadm kubectl -y

```



Hold versions:



```bash

sudo apt-mark hold kubelet kubeadm kubectl

```



---



## Initialize Kubernetes Cluster



Run on master node only:



```bash

sudo kubeadm init --pod-network-cidr=192.168.0.0/16

```



After successful initialization, save the worker join command.



Example:



```bash

sudo kubeadm join <MASTER_PRIVATE_IP>:6443 --token <TOKEN> --discovery-token-ca-cert-hash sha256:<HASH>

```



---



## Configure kubectl on Master



```bash

mkdir -p $HOME/.kube

```



```bash

sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config

```



```bash

sudo chown $(id -u):$(id -g) $HOME/.kube/config

```



Verify:



```bash

kubectl get nodes

```



---



## Install Calico Network Plugin



```bash

kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/master/manifests/calico.yaml

```



Check pods:



```bash

kubectl get pods -A

```



Wait until pods are running.



---



# 3. Kubernetes Worker Setup



SSH into worker server:



```bash

ssh -i your-key.pem ubuntu@<K8S_WORKER_PUBLIC_IP>

```



Update server:



```bash

sudo apt update

sudo apt upgrade -y

```



---



## Disable Swap



```bash

sudo swapoff -a

```



Edit fstab:



```bash

sudo nano /etc/fstab

```



Comment swap line if present.



---



## Load Kernel Modules



```bash

cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf

overlay

br_netfilter

EOF

```



```bash

sudo modprobe overlay

sudo modprobe br_netfilter

```



---



## Configure Kubernetes Networking



```bash

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf

net.bridge.bridge-nf-call-iptables = 1

net.bridge.bridge-nf-call-ip6tables = 1

net.ipv4.ip_forward = 1

EOF

```



```bash

sudo sysctl --system

```



---



## Install containerd



```bash

sudo apt install containerd -y

```



```bash

sudo mkdir -p /etc/containerd

containerd config default | sudo tee /etc/containerd/config.toml

```



Edit config:



```bash

sudo nano /etc/containerd/config.toml

```



Change:



```text

SystemdCgroup = false

```



to:



```text

SystemdCgroup = true

```



Restart containerd:



```bash

sudo systemctl restart containerd

sudo systemctl enable containerd

```



---



## Install kubeadm, kubelet and kubectl



```bash

sudo apt-get install -y apt-transport-https ca-certificates curl gpg

```



```bash

sudo mkdir -p /etc/apt/keyrings

```



```bash

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

```



```bash

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

```



```bash

sudo apt update

sudo apt install kubelet kubeadm kubectl -y

```



Hold versions:



```bash

sudo apt-mark hold kubelet kubeadm kubectl

```



---



## Join Worker Node to Cluster



Run the join command generated from master:



```bash

sudo kubeadm join <MASTER_PRIVATE_IP>:6443 --token <TOKEN> --discovery-token-ca-cert-hash sha256:<HASH>

```



Verify from master:



```bash

kubectl get nodes

```



Expected:



```text

k8s-master   Ready

k8s-worker   Ready

```



---



# 4. GitHub Repository Setup



Create GitHub repository:



```text

nodejs-cicd-demo

```



Clone repo on local system:



```bash

git clone https://github.com/<username>/nodejs-cicd-demo.git

```



Go inside repo:



```bash

cd nodejs-cicd-demo

```



Create project structure:



```bash

mkdir k8s

touch app.js package.json Dockerfile Jenkinsfile

touch k8s/deployment.yaml k8s/service.yaml

```



Project structure:



```text

nodejs-cicd-demo/

├── app.js

├── package.json

├── Dockerfile

├── Jenkinsfile

└── k8s/

    ├── deployment.yaml

    └── service.yaml

```



Push code:



```bash

git add .

git commit -m "Initial commit"

git branch -M main

git push origin main

```



For GitHub password, use Personal Access Token.



---



# 5. DockerHub Setup



Create DockerHub repository:



```text

nodejs-cicd-demo

```



This repository stores the Docker image pushed by Jenkins.



---



# 6. Jenkins Configuration



Open Jenkins:



```text

http://<JENKINS_PUBLIC_IP>:8080

```



Install suggested plugins.



Install additional plugins:



```text

Git Plugin

GitHub Plugin

Pipeline Plugin

Docker Pipeline Plugin

Credentials Plugin

```



---



## Add DockerHub Credentials



Go to:



```text

Manage Jenkins

→ Credentials

→ Global

→ Add Credentials

```



Type:



```text

Username with password

```



ID:



```text

dockerhub

```



Username:



```text

DockerHub username

```



Password:



```text

DockerHub password or token

```



---



## Add Kubernetes kubeconfig Credential



On k8s-master, run:



```bash

cat ~/.kube/config

```



Copy this file content and save it as a file named:



```text

kubeconfig

```



In Jenkins:



```text

Manage Jenkins

→ Credentials

→ Global

→ Add Credentials

```



Type:



```text

Secret file

```



Upload kubeconfig file.



ID:



```text

kubeconfig

```



---



# 7. Jenkins Pipeline Job Setup



Create new Jenkins job:



```text

New Item

→ Pipeline

```



Select:



```text

Pipeline script from SCM

```



SCM:



```text

Git

```



Repository URL:



```text

https://github.com/<username>/nodejs-cicd-demo.git

```



Branch:



```text

*/main

```



Script path:



```text

Jenkinsfile

```



Save.



---



# 8. GitHub Webhook Setup



Go to GitHub repository:



```text

Settings

→ Webhooks

→ Add webhook

```



Payload URL:



```text

http://<JENKINS_PUBLIC_IP>:8080/github-webhook/

```



Content type:



```text

application/json

```



Event:



```text

Just the push event

```



Save webhook.



---



# 9. Testing the Pipeline



Make any change in code:



```bash

git add .

git commit -m "Test CI/CD pipeline"

git push origin main

```



Expected flow:



```text

GitHub push

→ Webhook triggers Jenkins

→ Jenkins builds Docker image

→ Jenkins pushes image to DockerHub

→ Jenkins deploys app to Kubernetes

→ Application runs on worker node

```



---



# 10. Validation Commands



Run on k8s-master:



```bash

kubectl get nodes

```



```bash

kubectl get pods

```



```bash

kubectl get deployments

```



```bash

kubectl get svc

```



Check rollout:



```bash

kubectl rollout status deployment/nodejs-cicd-demo

```



Access application:



```text

http://<K8S_WORKER_PUBLIC_IP>:30080

```



Expected output:



```text

CI/CD Pipeline Working

```



---



# 11. Issues Faced and Solutions



## Issue 1: GitHub Webhook Not Working



Problem:



Jenkins port 8080 was allowed only for my IP address. Because of this, GitHub could not reach Jenkins.



Solution:



Allowed port 8080 in Jenkins Security Group from IPv4:



```text

0.0.0.0/0

```



---



## Issue 2: Jenkins Built-in Node Was Down



Problem:



Jenkins built-in node was offline because `/tmp` folder space was less than the Jenkins threshold of 1 GiB.



Solution:



Changed Jenkins node monitoring threshold from:



```text

1 GiB

```



to:



```text

500 MiB

```



Path:



```text

Manage Jenkins

→ Nodes

→ Built-In Node

→ Configure

```



---



## Issue 3: Pipeline Failed Due to Docker Permission



Problem:



Jenkins user did not have a proper home directory and did not have Docker permission.



Solution:



Created Jenkins home directory:



```bash

sudo mkdir -p /home/jenkins

sudo chown -R jenkins:jenkins /home/jenkins

```



Added Jenkins user to Docker group:



```bash

sudo usermod -aG docker jenkins

```



Restarted services:



```bash

sudo systemctl restart docker

sudo systemctl restart jenkins

```



Verified Docker access:



```bash

sudo -u jenkins docker ps

```



---



# 12. Security Best Practices



* DockerHub credentials are stored securely in Jenkins Credentials.

* Kubernetes kubeconfig is stored as a secret file in Jenkins.

* No passwords or tokens are hardcoded in Jenkinsfile.

* GitHub is used as the source of truth.

* Security groups are configured based on required ports only.



---

# Author



Jatin Shant



DevOps Engineer



Skills Used:



```text

AWS

Linux

Git

GitHub

Jenkins

Docker

DockerHub

Kubernetes

kubeadm

CI/CD

```



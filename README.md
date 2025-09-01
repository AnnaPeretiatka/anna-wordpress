# anna-WordPress with MariaDB on Kubernetes

This project deploys a WordPress application backed by MariaDB on a Kubernetes cluster, with full monitoring using Prometheus and Grafana. Everything is packaged with Helm charts for easy deployment.

---

## Project Structure

```
anna-wordpress/           
├── charts/                   
│   ├── anna-wordpress/       <- WordPress Helm chart
│   │   ├── templates/        <- Kubernetes templates for WordPress + MariaDB
│   │   │   ├── wordpress-deployment.yaml
│   │   │   ├── wordpress-service.yaml
│   │   │   ├── wordpress-ingress.yaml
│   │   │   ├── mariadb-statefulset.yaml
│   │   │   ├── mariadb-service.yaml
│   │   │   └── (other templates like NOTES.txt, _helpers.tpl)
│   │   ├── Chart.yaml
│   │   └── values.yaml
│   └── kube-prometheus-stack/ <- Prometheus + Grafana Helm chart
│       ├── templates/        
│       ├── Chart.yaml
│       └── values.yaml
└── README.md                
```

---

## 1. Prepare your EC2 Instance

Tested on **Ubuntu**, minimum t3.large, 60 GiB storage.
Ports to open: 80(HTTP-app), 3306(MariaDB), 3000 (Grafana), 9090(Prometheus), 8080(Kube-state-metrics), 9100(Node Exporter)

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl wget git unzip apt-transport-https ca-certificates gnupg lsb-release
```

### Install Docker

```bash
# Remove old Docker packages
for pkg in docker.io docker-doc docker-compose docker-compose-v2 podman-docker containerd runc; do sudo apt-get remove $pkg; done

# Add Docker's official GPG key
sudo apt-get update
sudo apt-get install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add Docker repository
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] \
https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}") stable" \
| sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
docker --version
```
Give user docker permisions
```bash
sudo usermod -aG docker ubuntu
exit
```
connect back 

### Install kubectl

```bash
curl -LO https://dl.k8s.io/release/v1.30.4/bin/linux/amd64/kubectl
chmod +x kubectl
sudo mv kubectl /usr/local/bin/
kubectl version --client
```

### Install Minikube

```bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
minikube version
```

Start Minikube:

```bash
minikube start --memory=6000 --cpus=2 --disk-size=50g --driver=docker
kubectl get nodes
kubectl get pods -A
```

### Install AWS CLI

```bash
sudo apt install -y unzip
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
aws --version
```

Configure AWS CLI:

```bash
aws configure
# Enter your Access Key, Secret, Region (us-east-1), and leave output format empty
```

---

## 2. Prepare Docker Images (Tal - you don't need to, my ecr is in files)

Pull WordPress and MariaDB images:

```bash
docker pull wordpress:latest
docker pull mariadb:10.6.4-focal
```

Push them to your ECR:

```bash
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin 992382545251.dkr.ecr.us-east-1.amazonaws.com
docker tag wordpress:latest 992382545251.dkr.ecr.us-east-1.amazonaws.com/anna-wordpress:wordpress
docker tag mariadb:10.6.4-focal 992382545251.dkr.ecr.us-east-1.amazonaws.com/anna-wordpress:mariadb
docker push 992382545251.dkr.ecr.us-east-1.amazonaws.com/anna-wordpress:wordpress
docker push 992382545251.dkr.ecr.us-east-1.amazonaws.com/anna-wordpress:mariadb
```

> change to your image on anna-wordpress/values.yaml

---

## 3. Install Helm & NGINX Ingress

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version

# NGINX Ingress Controller
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm install ingress-nginx ingress-nginx/ingress-nginx --namespace ingress-nginx --create-namespace
# check
kubectl get pods -n ingress-nginx
kubectl get svc -n ingress-nginx
```

## 4. Clone WordPress + MariaDB repo
```bash
git clone https://github.com/AnnaPeretiatka/anna-wordpress.git
cd anna-wordpress
```

## 5. Add Create & use an ECR pull secret (Tal - important)
```bash
# First authenticate Docker to ECR:
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin 992382545251.dkr.ecr.us-east-1.amazonaws.com

# Create a Kubernetes secret (regcred) for pulling from ECR:

kubectl create secret docker-registry regcred \
  --docker-server=992382545251.dkr.ecr.us-east-1.amazonaws.com \
  --docker-username=AWS \
  --docker-password="$(aws ecr get-login-password --region us-east-1)"
```

## 6. Deploy WordPress + MariaDB
Inside the repo:
```bash
cd charts/anna-wordpress
helm install anna-wordpress .
kubectl get pods
kubectl get svc
```
if neede to correct code, can use: helm upgrade --install anna-wordpress .

## 7. Deploy Monitoring (Prometheus + Grafana)
Monitoring Helm chart is in charts/kube-prometheus-stack, need to run:
```bash
cd ../kube-prometheus-stack
helm install kube-prometheus-stack . -n monitoring --create-namespace
kubectl get pods -n monitoring
```

## 8. Expose Services (Port Forwarding) 
### WordPress
```bash
nohup kubectl port-forward svc/anna-wordpress-wordpress 8080:80 --address 0.0.0.0 > port-forward.log 2>&1 &
```
Access it: http://<EC2_PUBLIC_IP>:8080/

### Grafana
```bash
nohup kubectl port-forward svc/kube-prometheus-stack-grafana -n monitoring 3000:80 --address 0.0.0.0 > grafana-port-forward.log 2>&1 &
```
Access it: http://<EC2_PUBLIC_IP>:3000/
user: admin
password: prom-operator

To check password, run:
```bash
kubectl get secret kube-prometheus-stack-grafana -n monitoring -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
```

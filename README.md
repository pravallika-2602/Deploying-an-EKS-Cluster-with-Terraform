# Deploying-an-EKS-Cluster-with-Terraform
# Deploying-an-EKS-Cluster-with-Terraform
 Deploying an EKS Cluster with Terraform and Deploying a BlogApp  using custom Helm and Nginx Ingress Controller


# 🚀 Deploying an EKS Cluster using Terraform, Helm & Nginx Ingress Controller

This guide walks through the deployment of an Amazon EKS (Elastic Kubernetes Service) cluster using **Terraform**, deployment of a **BlogApp** using **custom Helm charts**, and exposing it externally with the **Nginx Ingress Controller**.


## Prerequisite:

Install AWS CLI LATEST VERSION
INSTALL TERRAFORM 
INSTALL HELM
INSTALL KUBECTL 
CREATE A EC2 INSTANCE 
CREATE A USER AND GENERATE A ACCESS KEY ADD THE PERMISSIONS TO THAT USER

## Architectural Diagram:
<img width="359" alt="image" src="https://github.com/user-attachments/assets/72548455-e266-4b39-bb4c-f7592ddb7892" />

---

## 🛠️ Step 1: Create a Terraform Configuration to Deploy EKS Cluster

### 📁 `main.tf`

```hcl
provider "aws" {
  region = "ap-south-1"
}

data "aws_vpc" "default" {
  default = true
}

data "aws_subnets" "default" {
  filter {
    name   = "vpc-id"
    values = [data.aws_vpc.default.id]
  }
}

resource "aws_security_group" "eks_sg" {
  vpc_id = data.aws_vpc.default.id

  ingress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "eks-security-group"
  }
}

resource "aws_iam_role" "eks_role" {
  name = "eks-cluster-role"
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = "sts:AssumeRole"
      Effect = "Allow"
      Principal = {
        Service = "eks.amazonaws.com"
      }
    }]
  })
}

resource "aws_iam_role_policy_attachment" "eks_policy_attach" {
  role       = aws_iam_role.eks_role.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKSClusterPolicy"
}

resource "aws_eks_cluster" "eks_cluster" {
  name     = "my-eks-cluster"
  role_arn = aws_iam_role.eks_role.arn

  vpc_config {
    subnet_ids         = data.aws_subnets.default.ids
    security_group_ids = [aws_security_group.eks_sg.id]
  }
}

output "cluster_name" {
  value = aws_eks_cluster.eks_cluster.name
}

### 📁 `commands.sh`  
``` bash
terraform init
terraform validate
terraform plan
terraform apply -auto-approve


⚙️ Step 2: Configure Kubernetes Access
📁 kubeconfig.sh

aws eks update-kubeconfig --region ap-south-1 --name my-eks-cluster

📁 verify-cluster.sh

kubectl get nodes -A
kubectl get all -A

📦 Step 3: Create a Custom Helm Chart for BlogApp
📁 create-helm-chart.sh
\
helm create blogapp-chart

📁 blogapp-chart/templates/deployment.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}
  namespace: {{ .Values.namespace }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ .Values.appName }}
  template:
    metadata:
      labels:
        app: {{ .Values.appName }}
    spec:
      containers:
      - name: {{ .Values.appName }}
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        ports:
        - containerPort: {{ .Values.service.port }}

📁 blogapp-chart/templates/service.yaml

apiVersion: v1
kind: Service
metadata:
  name: {{ .Release.Name }}-svc
  namespace: {{ .Values.namespace }}
spec:
  selector:
    app: {{ .Values.appName }}
  ports:
    - protocol: TCP
      port: {{ .Values.service.port }}
      targetPort: {{ .Values.service.targetPort }}

📁 blogapp-chart/values.yaml

appName: blogapp
namespace: webapps
replicaCount: 1

image:
  repository: <your-repository>
  tag: "latest"

service:
  port: 8080
  targetPort: 8080

📁 install-helm-chart.sh

helm install blogapp ./blogapp-chart -n webapps
kubectl get all -n webapps

🌍 Step 4: Expose BlogApp with Nginx Ingress Controller

📁 install-ingress-controller.sh

helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm install nginx-ingress ingress-nginx/ingress-nginx --namespace ingress-nginx --create-namespace

📁 blogapp-chart/templates/ingress.yaml

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: blogapp-ingress
  namespace: webapps
  annotations:
    kubernetes.io/ingress.class: "nginx"
spec:
  rules:
  - host: test.example.com
    http:
      paths:
      - path: /blog
        pathType: Prefix
        backend:
          service:
            name: blogapp-svc
            port:
              number: 8080

📁 apply-ingress.sh

kubectl apply -f ingress.yaml
kubectl get ingress -n webapps

📁 hosts-configuration.txt

<INGRESS-EXTERNAL-IP> test.example.com
Then access the app at: http://test.example.com/blog

✅ Final Outcome
Your BlogApp is successfully:

Deployed on EKS via Terraform

Packaged and installed using Helm

Exposed externally through Nginx Ingress





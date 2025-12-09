# Enterprise 3-Tier Web Application on AWS EKS

Deploying a production-grade 3-tier application (React frontend, Flask backend, PostgreSQL RDS) on AWS EKS using `eksctl`, Helm, and Terraform.

## Application Architecture

- **Frontend**: React.js application served via Nginx.
- **Backend**: Python Flask API.
- **Database**: AWS RDS PostgreSQL (Private Subnet).
- **Networking**: AWS Application Load Balancer (ALB) via AWS Load Balancer Controller.
- **Security**: IAM Roles for Service Accounts (IRSA), OIDC, and Kubernetes Secrets.

## Prerequisites

- AWS CLI configured with administrator privileges.
- `eksctl`, `kubectl`, `helm` installed.
- Docker installed (for image building).

## 1. Cluster Provisioning

Create an EKS cluster with managed node groups using `eksctl`.

```bash
eksctl create cluster \
  --name nitish-eks-cluster \
  --region eu-west-1 \
  --version 1.31 \
  --nodegroup-name standard-workers \
  --node-type t3.medium \
  --nodes 2 \
  --nodes-min 1 \
  --nodes-max 3 \
  --managed

# Update kubeconfig
aws eks update-kubeconfig --name nitish-eks-cluster --region eu-west-1
```

## 2. Database Infrastructure (RDS)

Provision the PostgreSQL RDS instance in private subnets.

```bash
# Get VPC and Subnet details
VPC_ID=$(aws eks describe-cluster --name nitish-eks-cluster --region eu-west-1 --query "cluster.resourcesVpcConfig.vpcId" --output text)

# Create Subnet Group
aws rds create-db-subnet-group \
  --db-subnet-group-name nitish-postgres-private-subnet-group \
  --db-subnet-group-description "Private subnet group for Nitish PostgreSQL RDS" \
  --subnet-ids <subnet-id-1> <subnet-id-2> \
  --region eu-west-1

# Create Security Group
aws ec2 create-security-group --group-name postgressg --description "SG for RDS" --vpc-id $VPC_ID --region eu-west-1

# Authorize Ingress
NODE_SG=$(aws eks describe-cluster --name nitish-eks-cluster --region eu-west-1 --query "cluster.resourcesVpcConfig.securityGroupIds[0]" --output text)
aws ec2 authorize-security-group-ingress --group-id <sg-id> --protocol tcp --port 5432 --source-group $NODE_SG --region eu-west-1

# Create RDS Instance
aws rds create-db-instance \
  --db-instance-identifier nitish-postgres \
  --db-instance-class db.t3.small \
  --engine postgres \
  --master-username postgresadmin \
  --master-user-password YourStrongPassword123! \
  --db-subnet-group-name nitish-postgres-private-subnet-group \
  --no-publicly-accessible \
  --region eu-west-1
```

## 3. Containerization (Docker)

Build and push the application images to your container registry (ECR or Docker Hub).

```bash
# Authenticate (example for ECR)
aws ecr get-login-password --region eu-west-1 | docker login --username AWS --password-stdin <aws-account-id>.dkr.ecr.eu-west-1.amazonaws.com

# Build & Push Backend
docker build -t <your-registry>/3tier-backend:latest ./backend
docker push <your-registry>/3tier-backend:latest

# Build & Push Frontend
docker build -t <your-registry>/3tier-frontend:latest ./frontend
docker push <your-registry>/3tier-frontend:latest
```

## 4. Application Deployment

Deploy the Kubernetes manifests.

```bash
# Clone repository
git clone https://github.com/nitishrayalu/3-tier-app-eks.git
cd 3-tier-app-eks/k8s

# Apply Namespace & Configuration
kubectl apply -f namespace.yaml
kubectl apply -f configmap.yaml
kubectl apply -f secrets.yaml

# Create External Service for RDS
kubectl apply -f database-service.yaml

# Run Database Migrations
kubectl apply -f migration_job.yaml

# Deploy Backend & Frontend
kubectl apply -f backend.yaml
kubectl apply -f frontend.yaml
```

## 5. Ingress Setup (AWS Load Balancer Controller)

Configure IAM OIDC and install the ALB Controller.

```bash
# Associate OIDC Provider
eksctl utils associate-iam-oidc-provider --cluster nitish-eks-cluster --approve

# Create IAM Policy
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.11.0/docs/install/iam_policy.json
aws iam create-policy --policy-name AWSLoadBalancerControllerIAMPolicy --policy-document file://iam_policy.json

# Create Service Account
eksctl create iamserviceaccount \
  --cluster=nitish-eks-cluster \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn=arn:aws:iam::<aws-account-id>:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve

# Install Controller via Helm
helm repo add eks https://aws.github.io/eks-charts
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=nitish-eks-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller

# Deploy Ingress
kubectl apply -f ingress.yaml
```

## 6. DNS Verification

Configuring Route53 alias for the ALB.

```bash
# Get ALB DNS
kubectl get ingress -n 3-tier-app-eks

# Create Route53 Record
aws route53 change-resource-record-sets --hosted-zone-id <zone-id> --change-batch file://dns-setup.json
```

## Troubleshooting

- **Database Connection**: Check `kubectl logs` on backend pods. Ensure RDS Security Group allows traffic from EKS Node Security Group.
- **Ingress**: Check `kubectl logs -n kube-system -l app.kubernetes.io/name=aws-load-balancer-controller`. Ensure subnets are tagged `kubernetes.io/role/elb: 1` (public) or `kubernetes.io/role/internal-elb: 1` (private).

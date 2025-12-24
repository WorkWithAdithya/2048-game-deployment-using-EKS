# 2048 Game Deployment on AWS EKS

A guide to deploying the popular 2048 game on Amazon Elastic Kubernetes Service (EKS) using Fargate and Application Load Balancer with Ingress Controller.

## 📋 Overview

This project demonstrates how to deploy a containerized application (2048 game) on AWS EKS with a managed Kubernetes cluster. It uses AWS Fargate for serverless container management and an Application Load Balancer through Kubernetes Ingress for cost-effective traffic routing.

## 🎯 Why EKS?

- **Managed Control Plane**: AWS handles the complexity of setting up and maintaining the Kubernetes master/control plane
- **Reduced Operational Overhead**: Eliminates tedious setup, error-prone configurations, and difficult debugging
- **High Availability**: Built-in reliability and fault tolerance

## 🏗️ Architecture

### Components Used

1. **EKS Cluster**: Managed Kubernetes control plane
2. **AWS Fargate**: Serverless compute for running containers (no EC2 management needed)
3. **Ingress Controller**: AWS Load Balancer Controller for traffic management
4. **Application Load Balancer**: Routes external traffic to pods in private VPC

### How Ingress Works

```
User Request → ALB (Public Subnet) → Ingress Controller → Kubernetes Service → Pods (Private VPC)
```

- Pods run in a private VPC (not directly accessible)
- Ingress resource defines routing rules
- Ingress Controller provisions an ALB in the public subnet
- Users access the application via the ALB's DNS name

### Why Ingress Over LoadBalancer Service?

Using Kubernetes LoadBalancer service type creates one ALB per service, resulting in high costs. Ingress allows multiple services to share a single ALB, significantly reducing infrastructure costs.

## 🔧 Prerequisites

Install the following tools on your local machine:

1. **kubectl** - Kubernetes command-line tool
   ```
   https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/
   ```

2. **eksctl** - EKS cluster management CLI
   ```
   https://docs.aws.amazon.com/eks/latest/eksctl/installation.html
   ```

3. **AWS CLI** - AWS command-line interface
   ```
   https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html
   ```

4. **Helm** (v3+) - Kubernetes package manager

## ⚙️ Setup Instructions

### Step 1: Configure AWS Credentials

```bash
aws configure
```
Enter your AWS Access Key ID and Secret Access Key when prompted.

### Step 2: Create EKS Cluster

```bash
eksctl create cluster --name demo-cluster --region us-east-1 --fargate
```

This command automatically creates:
- VPC and subnets
- EKS cluster with Fargate profile
- Required security groups and IAM roles

**Note**: Using CLI simplifies the process compared to manual AWS Console setup.

### Step 3: Update Kubeconfig

```bash
aws eks update-kubeconfig --name demo-cluster --region us-east-1
```

This configures kubectl to communicate with your new EKS cluster.

### Step 4: Create Fargate Profile

```bash
eksctl create fargateprofile \
    --cluster demo-cluster \
    --region us-east-1 \
    --name alb-sample-app \
    --namespace game-2048
```

### Step 5: Deploy 2048 Game Application

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/examples/2048/2048_full.yaml
```

This deploys the application pods, service, and ingress resources.

### Step 6: Setup IAM OIDC Provider

```bash
eksctl utils associate-iam-oidc-provider --cluster demo-cluster --approve
```

This allows Kubernetes service accounts to assume IAM roles for accessing AWS resources.

### Step 7: Create IAM Policy for Load Balancer Controller

```bash
# Download the IAM policy
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.11.0/docs/install/iam_policy.json

# Create the policy
aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam_policy.json
```

### Step 8: Create IAM Service Account

```bash
eksctl create iamserviceaccount \
  --cluster=demo-cluster \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn=arn:aws:iam::<YOUR-ACCOUNT-ID>:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve
```

**Important**: Replace `<YOUR-ACCOUNT-ID>` with your AWS account ID.

### Step 9: Install AWS Load Balancer Controller using Helm

```bash
# Add the EKS Helm repository
helm repo add eks https://aws.github.io/eks-charts

# Update the repository
helm repo update

# Install the AWS Load Balancer Controller
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=demo-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=us-east-1 \
  --set vpcId=<YOUR-VPC-ID>
```

**Note**: Replace `<YOUR-VPC-ID>` with your cluster's VPC ID (find it in the EKS console or AWS VPC dashboard).

### Step 10: Verify Deployment

```bash
kubectl get deployment -n kube-system aws-load-balancer-controller
```

Check that the controller is running successfully.

### Step 11: Get Application URL

```bash
kubectl get ingress -n game-2048
```

Look for the ALB address in the output. You can also find it in:
**AWS Console → EC2 → Load Balancers → Copy DNS name**

Access the application by pasting the DNS name in your browser!

## 🎮 Accessing the Game

Once deployed, navigate to the Application Load Balancer's DNS name in your web browser to play the 2048 game.

## 🧹 Cleanup

To avoid ongoing AWS charges, delete the resources when done:

```bash
# Delete the cluster (this will delete all associated resources)
eksctl delete cluster --name demo-cluster --region us-east-1
```

## 📚 Additional Resources

- [AWS EKS Documentation](https://docs.aws.amazon.com/eks/)
- [AWS Load Balancer Controller](https://kubernetes-sigs.github.io/aws-load-balancer-controller/)
- [Kubernetes Ingress Documentation](https://kubernetes.io/docs/concepts/services-networking/ingress/)

## 🤝 Contributing

Feel free to submit issues or pull requests for improvements!

## 📝 License

This project is for educational purposes.

---

**Happy Gaming! 🎮**

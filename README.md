# Java E-Commerce Backend DevOps Project

A complete end-to-end DevOps pipeline for a Spring Boot e-commerce backend application, demonstrating modern cloud-native practices with AWS EKS, Jenkins CI/CD, and comprehensive monitoring.

## 🏗️ Architecture Overview

This project implements a production-ready e-commerce backend with:
- **Spring Boot REST API** with Product management
- **Containerized deployment** using Docker
- **AWS EKS Kubernetes cluster** for orchestration
- **Jenkins CI/CD pipeline** for automated deployments
- **Prometheus & Grafana** for monitoring and observability
- **Infrastructure as Code** using Terraform
- **Application Load Balancer** for traffic distribution

## 📋 Table of Contents

- [Prerequisites](#prerequisites)
- [Project Structure](#project-structure)
- [Quick Start](#quick-start)
- [Infrastructure Setup](#infrastructure-setup)
- [Application Deployment](#application-deployment)
- [CI/CD Pipeline](#cicd-pipeline)
- [Monitoring Setup](#monitoring-setup)
- [API Documentation](#api-documentation)
- [Local Development](#local-development)
- [Troubleshooting](#troubleshooting)
- [Clean Up](#clean-up)

## 🔧 Prerequisites

### Required Tools
- **AWS CLI** v2.x configured with appropriate permissions
- **Docker** v20.x or later
- **kubectl** v1.28 or later
- **Terraform** v1.6 or later
- **Helm** v3.x
- **Java 17** JDK
- **Maven** 3.8+
- **Git**

### AWS Permissions Required
- EKS full access
- EC2 full access
- ECR full access
- VPC full access
- IAM role creation
- Load Balancer management

### EC2 Instance Requirements
- **Instance Type**: t3.large (minimum for Jenkins + development)
- **Storage**: 20GB GP2
- **OS**: Ubuntu 22.04 LTS
- **Security Groups**: Ports 8080, 8081, 3000, 9090 open

## 📁 Project Structure

```
ecommerce-backend/
├── src/
│   └── main/
│       ├── java/com/ecommerce/
│       │   ├── ECommerceApplication.java
│       │   ├── controller/ProductController.java
│       │   ├── model/Product.java
│       │   ├── repository/ProductRepository.java
│       │   └── service/ProductService.java
│       └── resources/
│           └── application.properties
├── terraform/
│   ├── main.tf                    # Main infrastructure
│   ├── variables.tf               # Configuration variables
│   ├── outputs.tf                 # Output values
│   └── versions.tf                # Provider versions
├── k8s/
│   ├── deployment.yaml            # Kubernetes deployment
│   ├── service.yaml               # Kubernetes service
│   ├── ingress.yaml               # ALB ingress configuration
│   └── monitoring/
│       ├── servicemonitor.yaml    # Prometheus service monitor
│       ├── monitoring-ingress.yaml # Monitoring ingress
│       └── prometheus.yml         # Local Prometheus config
├── jenkins/
│   └── Jenkinsfile                # CI/CD pipeline definition
├── Dockerfile                     # Container image definition
├── docker-compose.yml             # Local development stack
├── pom.xml                        # Maven dependencies
└── README.md
```

## 🚀 Quick Start

### 1. Environment Setup

```bash
# Clone the repository
git clone <your-repo-url>
cd ecommerce-backend

# Install prerequisites on Ubuntu EC2
sudo apt update && sudo apt upgrade -y
sudo apt install -y openjdk-17-jdk maven docker.io git curl wget unzip

# Configure AWS CLI
aws configure
# Enter your AWS Access Key, Secret Key, and set region to us-west-1
```

### 2. Infrastructure Deployment

```bash
# Initialize and deploy infrastructure
cd terraform
terraform init
terraform plan
terraform apply -auto-approve

# Configure kubectl
aws eks update-kubeconfig --region us-west-1 --name ecommerce-eks-cluster

# Verify cluster access
kubectl get nodes
```

### 3. Application Build and Deploy

```bash
# Build the application
mvn clean package

# Build Docker image
docker build -t ecommerce-backend:latest .

# Push to ECR
ECR_REPO=$(terraform output -raw ecr_repository_url)
aws ecr get-login-password --region us-west-1 | docker login --username AWS --password-stdin $ECR_REPO
docker tag ecommerce-backend:latest $ECR_REPO:latest
docker push $ECR_REPO:latest

# Deploy to Kubernetes
kubectl apply -f k8s/deployment.yaml
kubectl apply -f k8s/service.yaml
kubectl apply -f k8s/ingress.yaml
```

## 🏛️ Infrastructure Setup

### AWS Resources Created

The Terraform configuration creates:

- **VPC** with public/private subnets across 2 AZs
- **EKS Cluster** with managed node groups
- **ECR Repository** for container images
- **Security Groups** with appropriate rules
- **IAM Roles** for EKS cluster and nodes
- **NAT Gateways** for private subnet internet access

### Key Configuration

| Component | Configuration |
|-----------|---------------|
| EKS Version | 1.28 |
| Node Group | t3.medium instances |
| Scaling | 1-3 nodes (desired: 2) |
| VPC CIDR | 10.0.0.0/16 |
| Region | us-west-1 |

## 🔄 CI/CD Pipeline

### Jenkins Pipeline Stages

1. **Checkout** - Clone source code
2. **Build Maven** - Compile and package application
3. **Run Tests** - Execute unit tests
4. **Build Docker Image** - Create container image
5. **Push to ECR** - Upload image to registry
6. **Deploy to EKS** - Update Kubernetes deployment
7. **Verify Deployment** - Health checks and validation

### Pipeline Configuration

```bash
# Install Jenkins
sudo apt install -y openjdk-11-jdk
curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key | \
  sudo tee /usr/share/keyrings/jenkins-keyring.asc > /dev/null
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian-stable binary/ | \
  sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt-get update && sudo apt-get install -y jenkins

# Configure Jenkins
sudo systemctl enable jenkins
sudo systemctl start jenkins
sudo usermod -aG docker jenkins
```

Access Jenkins at: `http://YOUR_EC2_IP:8081`

## 📊 Monitoring Setup

### Prometheus & Grafana Installation

```bash
# Create monitoring namespace
kubectl create namespace monitoring

# Install Prometheus stack
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --set prometheus.prometheusSpec.serviceMonitorSelectorNilUsesHelmValues=false \
  --set grafana.adminPassword=admin123

# Apply monitoring configuration
kubectl apply -f k8s/monitoring/servicemonitor.yaml
kubectl apply -f k8s/monitoring/monitoring-ingress.yaml
```

### Access URLs

- **Grafana**: `http://MONITORING_ALB_URL/grafana` (admin/admin123)
- **Prometheus**: `http://MONITORING_ALB_URL/prometheus`

### Key Metrics Monitored

- HTTP request rates and response times
- JVM memory and CPU usage
- Database connection pool status
- Application health status
- Kubernetes pod metrics

## 📚 API Documentation

### Base URL
```
Production: http://ALB_URL/api
Local: http://localhost:8080/api
```

### Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/products` | List all products |
| GET | `/products/{id}` | Get product by ID |
| POST | `/products` | Create new product |
| PUT | `/products/{id}` | Update product |
| DELETE | `/products/{id}` | Delete product |

### Example Requests

```bash
# Create a product
curl -X POST http://ALB_URL/api/products \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Gaming Laptop",
    "description": "High-performance gaming laptop",
    "price": 1299.99,
    "stock": 15
  }'

# Get all products
curl http://ALB_URL/api/products

# Get product by ID
curl http://ALB_URL/api/products/1
```

### Health Endpoints

- **Health Check**: `/actuator/health`
- **Metrics**: `/actuator/prometheus`
- **Application Info**: `/actuator/info`

## 💻 Local Development

### Using Docker Compose

```bash
# Start local development environment
docker-compose up -d

# Access services
# Application: http://localhost:8080
# Prometheus: http://localhost:9090
# Grafana: http://localhost:3000 (admin/admin123)

# Stop services
docker-compose down
```

### Development Profile

The application uses different configurations for different environments:

- **Local**: H2 in-memory database
- **Production**: Optimized for Kubernetes deployment

## 🐛 Troubleshooting

### Common Issues

#### ALB Not Creating
```bash
# Check ALB controller
kubectl get deployment -n kube-system aws-load-balancer-controller
kubectl logs -n kube-system deployment/aws-load-balancer-controller

# Verify subnet tags
aws ec2 describe-subnets --filters "Name=tag:kubernetes.io/role/elb,Values=1"
```

#### Application Not Starting
```bash
# Check pod logs
kubectl get pods -l app=ecommerce-backend
kubectl logs -l app=ecommerce-backend

# Check service and ingress
kubectl describe svc ecommerce-backend-service
kubectl describe ingress ecommerce-backend-ingress
```

#### Jenkins Build Failures
```bash
# Check Jenkins logs
sudo journalctl -u jenkins -f

# Verify Docker permissions
sudo usermod -aG docker jenkins
sudo systemctl restart jenkins

# Test AWS credentials
aws sts get-caller-identity
```

#### Monitoring Issues
```bash
# Check ServiceMonitor
kubectl get servicemonitor -n monitoring
kubectl describe servicemonitor ecommerce-backend-monitor -n monitoring

# Verify Prometheus targets
# Access Prometheus UI and check Status > Targets
```

### Useful Commands

```bash
# Get application URL
kubectl get ingress ecommerce-backend-ingress -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'

# Scale deployment
kubectl scale deployment ecommerce-backend --replicas=3

# Rolling update
kubectl set image deployment/ecommerce-backend ecommerce-backend=NEW_IMAGE:TAG

# View resource usage
kubectl top nodes
kubectl top pods
```

## 🧹 Clean Up

### Delete Kubernetes Resources
```bash
kubectl delete -f k8s/
kubectl delete namespace monitoring
helm uninstall prometheus -n monitoring
```

### Destroy Infrastructure
```bash
cd terraform
terraform destroy -auto-approve
```

### Clean ECR Repository
```bash
aws ecr list-images --repository-name ecommerce-backend --query 'imageIds[*]' --output text | \
while read imageId; do
  aws ecr batch-delete-image --repository-name ecommerce-backend --image-ids imageDigest=$imageId
done
```

## 🚀 Project Extensions

Consider these enhancements for production use:

1. **Database Integration**: Replace H2 with Amazon RDS PostgreSQL
2. **Security**: Implement Spring Security with JWT authentication
3. **Caching**: Add Redis for application caching
4. **Service Mesh**: Implement Istio for advanced traffic management
5. **GitOps**: Use ArgoCD for GitOps-based deployments
6. **Multi-Environment**: Create separate environments (dev/staging/prod)
7. **Load Testing**: Add performance testing with tools like JMeter
8. **Backup Strategy**: Implement automated backup and disaster recovery

## 📄 License

This project is licensed under the MIT License - see the LICENSE file for details.

## 🤝 Contributing

1. Fork the repository
2. Create a feature branch
3. Commit your changes
4. Push to the branch
5. Create a Pull Request

## 📞 Support

For questions and support:
- Create an issue in the repository
- Check the troubleshooting section
- Review AWS and Kubernetes documentation

---

**Project Status**: Production Ready ✅  
**Last Updated**: June 2025  
**Maintainer**: DevOps Team# ecommerce-backend

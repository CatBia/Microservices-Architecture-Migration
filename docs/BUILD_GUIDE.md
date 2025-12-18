# Movister Parallel Porter - Step-by-Step Build Guide

This comprehensive guide will walk you through building the Movister Parallel Porter microservices architecture from scratch.

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Project Setup](#project-setup)
3. [Infrastructure Setup](#infrastructure-setup)
4. [Database Setup](#database-setup)
5. [Service Development](#service-development)
6. [Containerization](#containerization)
7. [Kubernetes Deployment](#kubernetes-deployment)
8. [CI/CD Pipeline](#cicd-pipeline)
9. [Monitoring & Observability](#monitoring--observability)
10. [Security Configuration](#security-configuration)
11. [Testing & Validation](#testing--validation)

---

## Prerequisites

### 1. Install Required Tools

```bash
# Install Docker and Docker Compose
curl -fsSL https://get.docker.com -o get-docker.sh
sh get-docker.sh
sudo usermod -aG docker $USER

# Install kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# Install Helm
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Install Terraform
wget https://releases.hashicorp.com/terraform/1.5.0/terraform_1.5.0_linux_amd64.zip
unzip terraform_1.5.0_linux_amd64.zip
sudo mv terraform /usr/local/bin/

# Install AWS CLI
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

# Install Node.js (for API Gateway and services)
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt-get install -y nodejs

# Install Go (for gRPC services)
wget https://go.dev/dl/go1.21.0.linux-amd64.tar.gz
sudo tar -C /usr/local -xzf go1.21.0.linux-amd64.tar.gz
export PATH=$PATH:/usr/local/go/bin
```

### 2. Configure AWS Credentials LOCALLY

```bash
aws configure
# Enter your AWS Access Key ID
# Enter your AWS Secret Access Key
# Enter your default region (e.g., us-east-1)
# Enter default output format (json)
```

### 3. Verify Installations

```bash
docker --version
kubectl version --client
helm version
terraform version
aws --version
node --version
go version
```

---

## Project Setup

### Step 1: Clone and Initialize Repository

```bash
# Clone the repository
git clone https://github.com/CatBia/Microservices-Architecture-Migration.git
cd Microservices-Architecture-Migration

# Create project structure`
mkdir -p terraform/modules/{eks,vpc,rds,security}
mkdir -p terraform/environments/{dev,staging,production}
mkdir -p kubernetes/{base,overlays/{dev,staging,production},helm-charts/{service-template,infrastructure}}
mkdir -p services/{api-gateway,user-service,product-service,order-service,payment-service,inventory-service,notification-service,search-service}
mkdir -p shared/{proto,libs,config}
mkdir -p scripts
mkdir -p docs/{architecture,deployment,api}
```

### Step 2: Create Base Configuration Files

```bash
# Create Makefile
cat > Makefile << 'EOF'
.PHONY: dev build test deploy

dev:
	docker-compose up -d

build:
	@echo "Building all services..."
	@for service in services/*/; do \
		echo "Building $$service"; \
		cd $$service && docker build -t $$(basename $$service):latest . && cd ../..; \
	done

test:
	@echo "Running tests..."
	@for service in services/*/; do \
		echo "Testing $$service"; \
		cd $$service && npm test || go test ./... && cd ../..; \
	done

deploy:
	@echo "Deploying to Kubernetes..."
	kubectl apply -k kubernetes/overlays/dev
EOF

# Create .gitignore
cat > .gitignore << 'EOF'
# Terraform
*.tfstate
*.tfstate.*
.terraform/
.terraform.lock.hcl

# Docker
*.log
.env

# Kubernetes
*.kubeconfig

# IDE
.vscode/
.idea/
*.swp
*.swo

# Dependencies
node_modules/
vendor/
EOF
```

---

## Infrastructure Setup

### Step 3: Create Terraform VPC Module

```bash
# Create terraform/modules/vpc/main.tf
cat > terraform/modules/vpc/main.tf << 'EOF'
variable "region" {
  description = "AWS region"
  type        = string
}

variable "vpc_cidr" {
  description = "CIDR block for VPC"
  type        = string
  default     = "10.0.0.0/16"
}

variable "availability_zones" {
  description = "Availability zones"
  type        = list(string)
}

resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = {
    Name = "movister-vpc"
  }
}

resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id

  tags = {
    Name = "movister-igw"
  }
}

resource "aws_subnet" "public" {
  count             = length(var.availability_zones)
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 8, count.index)
  availability_zone = var.availability_zones[count.index]

  map_public_ip_on_launch = true

  tags = {
    Name = "movister-public-subnet-${count.index + 1}"
  }
}

resource "aws_subnet" "private" {
  count             = length(var.availability_zones)
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 8, count.index + 10)
  availability_zone = var.availability_zones[count.index]

  tags = {
    Name = "movister-private-subnet-${count.index + 1}"
  }
}

resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }

  tags = {
    Name = "movister-public-rt"
  }
}

resource "aws_route_table_association" "public" {
  count          = length(aws_subnet.public)
  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}

output "vpc_id" {
  value = aws_vpc.main.id
}

output "public_subnet_ids" {
  value = aws_subnet.public[*].id
}

output "private_subnet_ids" {
  value = aws_subnet.private[*].id
}
EOF
```

### Step 4: Create EKS Module

```bash
# Create terraform/modules/eks/main.tf
cat > terraform/modules/eks/main.tf << 'EOF'
variable "cluster_name" {
  description = "EKS cluster name"
  type        = string
}

variable "vpc_id" {
  description = "VPC ID"
  type        = string
}

variable "subnet_ids" {
  description = "Subnet IDs"
  type        = list(string)
}

variable "node_group_instance_types" {
  description = "Instance types for node group"
  type        = list(string)
  default     = ["t3.medium"]
}

resource "aws_eks_cluster" "main" {
  name     = var.cluster_name
  role_arn = aws_iam_role.cluster.arn
  version  = "1.27"

  vpc_config {
    subnet_ids = var.subnet_ids
  }

  depends_on = [
    aws_iam_role_policy_attachment.cluster_AmazonEKSClusterPolicy,
  ]
}

resource "aws_iam_role" "cluster" {
  name = "${var.cluster_name}-cluster-role"

  assume_role_policy = jsonencode({
    Statement = [{
      Action = "sts:AssumeRole"
      Effect = "Allow"
      Principal = {
        Service = "eks.amazonaws.com"
      }
    }]
    Version = "2012-10-17"
  })
}

resource "aws_iam_role_policy_attachment" "cluster_AmazonEKSClusterPolicy" {
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKSClusterPolicy"
  role       = aws_iam_role.cluster.name
}

resource "aws_eks_node_group" "main" {
  cluster_name    = aws_eks_cluster.main.name
  node_group_name = "${var.cluster_name}-node-group"
  node_role_arn   = aws_iam_role.nodes.arn
  subnet_ids      = var.subnet_ids
  instance_types  = var.node_group_instance_types

  scaling_config {
    desired_size = 2
    max_size     = 4
    min_size     = 1
  }

  depends_on = [
    aws_iam_role_policy_attachment.nodes_AmazonEKSWorkerNodePolicy,
    aws_iam_role_policy_attachment.nodes_AmazonEKS_CNI_Policy,
    aws_iam_role_policy_attachment.nodes_AmazonEC2ContainerRegistryReadOnly,
  ]
}

resource "aws_iam_role" "nodes" {
  name = "${var.cluster_name}-node-role"

  assume_role_policy = jsonencode({
    Statement = [{
      Action = "sts:AssumeRole"
      Effect = "Allow"
      Principal = {
        Service = "ec2.amazonaws.com"
      }
    }]
    Version = "2012-10-17"
  })
}

resource "aws_iam_role_policy_attachment" "nodes_AmazonEKSWorkerNodePolicy" {
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy"
  role       = aws_iam_role.nodes.name
}

resource "aws_iam_role_policy_attachment" "nodes_AmazonEKS_CNI_Policy" {
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy"
  role       = aws_iam_role.nodes.name
}

resource "aws_iam_role_policy_attachment" "nodes_AmazonEC2ContainerRegistryReadOnly" {
  policy_arn = "arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly"
  role       = aws_iam_role.nodes.name
}

output "cluster_id" {
  value = aws_eks_cluster.main.id
}

output "cluster_endpoint" {
  value = aws_eks_cluster.main.endpoint
}
EOF
```

### Step 5: Create Development Environment Configuration

```bash
# Create terraform/environments/dev/main.tf
cat > terraform/environments/dev/main.tf << 'EOF'
terraform {
  required_version = ">= 1.0"
  
  backend "s3" {
    bucket = "movister-terraform-state"
    key    = "dev/terraform.tfstate"
    region = "us-east-1"
  }
}

provider "aws" {
  region = "us-east-1"
}

module "vpc" {
  source = "../../modules/vpc"
  
  region            = "us-east-1"
  vpc_cidr          = "10.0.0.0/16"
  availability_zones = ["us-east-1a", "us-east-1b"]
}

module "eks" {
  source = "../../modules/eks"
  
  cluster_name = "movister-dev"
  vpc_id       = module.vpc.vpc_id
  subnet_ids   = module.vpc.private_subnet_ids
}
EOF

# Initialize and apply Terraform
cd terraform/environments/dev
terraform init
terraform plan
terraform apply
```

### Step 6: Configure kubectl for EKS

```bash
# Update kubeconfig
aws eks update-kubeconfig --name movister-dev --region us-east-1

# Verify connection
kubectl get nodes
```

---

## Database Setup

### Step 7: Create RDS PostgreSQL Module

```bash
# Create terraform/modules/rds/main.tf
cat > terraform/modules/rds/main.tf << 'EOF'
variable "db_name" {
  description = "Database name"
  type        = string
}

variable "db_username" {
  description = "Database master username"
  type        = string
}

variable "db_password" {
  description = "Database master password"
  type        = string
  sensitive   = true
}

variable "subnet_ids" {
  description = "Subnet IDs for RDS"
  type        = list(string)
}

variable "vpc_id" {
  description = "VPC ID"
  type        = string
}

resource "aws_db_subnet_group" "main" {
  name       = "${var.db_name}-subnet-group"
  subnet_ids = var.subnet_ids

  tags = {
    Name = "${var.db_name}-subnet-group"
  }
}

resource "aws_security_group" "rds" {
  name        = "${var.db_name}-sg"
  description = "Security group for RDS"
  vpc_id      = var.vpc_id

  ingress {
    from_port   = 5432
    to_port     = 5432
    protocol    = "tcp"
    cidr_blocks = ["10.0.0.0/16"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "${var.db_name}-sg"
  }
}

resource "aws_db_instance" "main" {
  identifier             = var.db_name
  engine                 = "postgres"
  engine_version         = "14.9"
  instance_class         = "db.t3.medium"
  allocated_storage      = 20
  storage_encrypted      = true
  db_name                = var.db_name
  username               = var.db_username
  password               = var.db_password
  db_subnet_group_name   = aws_db_subnet_group.main.name
  vpc_security_group_ids = [aws_security_group.rds.id]
  multi_az               = true
  backup_retention_period = 7
  skip_final_snapshot    = true

  tags = {
    Name = var.db_name
  }
}

output "db_endpoint" {
  value = aws_db_instance.main.endpoint
}

output "db_name" {
  value = aws_db_instance.main.db_name
}
EOF
```

### Step 8: Set Up Databases

```bash
# Add RDS to dev environment
# Edit terraform/environments/dev/main.tf to include RDS module

# Create MongoDB and Redis using Kubernetes (for local/dev)
# We'll use Helm charts for these
```

---

## Service Development

### Step 9: Create API Gateway Service

```bash
cd services/api-gateway

# Initialize Node.js project
npm init -y
npm install apollo-server graphql express cors dotenv
npm install --save-dev @types/node typescript ts-node nodemon

# Create package.json scripts
cat > package.json << 'EOF'
{
  "name": "api-gateway",
  "version": "1.0.0",
  "scripts": {
    "start": "node dist/index.js",
    "dev": "nodemon --exec ts-node src/index.ts",
    "build": "tsc",
    "test": "jest"
  },
  "dependencies": {
    "apollo-server": "^3.12.0",
    "graphql": "^16.8.1",
    "express": "^4.18.2",
    "cors": "^2.8.5",
    "dotenv": "^16.3.1"
  },
  "devDependencies": {
    "@types/node": "^20.5.0",
    "@types/express": "^4.17.17",
    "typescript": "^5.1.6",
    "ts-node": "^10.9.1",
    "nodemon": "^3.0.1"
  }
}
EOF

# Create TypeScript config
cat > tsconfig.json << 'EOF'
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "commonjs",
    "lib": ["ES2020"],
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules"]
}
EOF

# Create source file
mkdir -p src
cat > src/index.ts << 'EOF'
import { ApolloServer } from 'apollo-server';
import express from 'express';
import cors from 'cors';
import dotenv from 'dotenv';

dotenv.config();

const app = express();
app.use(cors());

const typeDefs = `
  type Query {
    health: String
  }
`;

const resolvers = {
  Query: {
    health: () => 'API Gateway is healthy',
  },
};

const server = new ApolloServer({
  typeDefs,
  resolvers,
});

const PORT = process.env.PORT || 8080;

server.listen({ port: PORT }).then(({ url }) => {
  console.log(`🚀 API Gateway ready at ${url}`);
});

// REST endpoint
app.get('/health', (req, res) => {
  res.json({ status: 'healthy', service: 'api-gateway' });
});

app.listen(8081, () => {
  console.log(`REST API listening on port 8081`);
});
EOF

# Create Dockerfile
cat > Dockerfile << 'EOF'
FROM node:18-alpine

WORKDIR /app

COPY package*.json ./
RUN npm ci --only=production

COPY . .
RUN npm run build

EXPOSE 8080 8081

CMD ["npm", "start"]
EOF

# Create .dockerignore
cat > .dockerignore << 'EOF'
node_modules
npm-debug.log
.env
.git
.gitignore
README.md
EOF
```

### Step 10: Create User Service (gRPC + REST)

```bash
cd ../user-service

# Initialize Go module
go mod init user-service

# Install dependencies
go get google.golang.org/grpc
go get google.golang.org/protobuf
go get github.com/gin-gonic/gin
go get github.com/lib/pq

# Create proto directory
mkdir -p proto

# Create user.proto
cat > proto/user.proto << 'EOF'
syntax = "proto3";

package user;

service UserService {
  rpc GetUser(GetUserRequest) returns (User);
  rpc CreateUser(CreateUserRequest) returns (User);
  rpc UpdateUser(UpdateUserRequest) returns (User);
}

message GetUserRequest {
  string user_id = 1;
}

message CreateUserRequest {
  string email = 1;
  string name = 2;
}

message UpdateUserRequest {
  string user_id = 1;
  string name = 2;
}

message User {
  string id = 1;
  string email = 2;
  string name = 3;
  string created_at = 4;
}
EOF

# Generate Go code from proto
go install google.golang.org/protobuf/cmd/protoc-gen-go@latest
go install google.golang.org/grpc/cmd/protoc-gen-go-grpc@latest
protoc --go_out=. --go-grpc_out=. proto/user.proto

# Create main.go
cat > main.go << 'EOF'
package main

import (
	"context"
	"database/sql"
	"log"
	"net"
	"os"

	"github.com/gin-gonic/gin"
	_ "github.com/lib/pq"
	"google.golang.org/grpc"

	pb "user-service/proto"
)

type server struct {
	pb.UnimplementedUserServiceServer
	db *sql.DB
}

func (s *server) GetUser(ctx context.Context, req *pb.GetUserRequest) (*pb.User, error) {
	var user pb.User
	err := s.db.QueryRow("SELECT id, email, name, created_at FROM users WHERE id = $1", req.UserId).
		Scan(&user.Id, &user.Email, &user.Name, &user.CreatedAt)
	if err != nil {
		return nil, err
	}
	return &user, nil
}

func (s *server) CreateUser(ctx context.Context, req *pb.CreateUserRequest) (*pb.User, error) {
	var user pb.User
	err := s.db.QueryRow(
		"INSERT INTO users (email, name) VALUES ($1, $2) RETURNING id, email, name, created_at",
		req.Email, req.Name,
	).Scan(&user.Id, &user.Email, &user.Name, &user.CreatedAt)
	if err != nil {
		return nil, err
	}
	return &user, nil
}

func main() {
	db, err := sql.Open("postgres", os.Getenv("DATABASE_URL"))
	if err != nil {
		log.Fatal(err)
	}
	defer db.Close()

	// Create table if not exists
	db.Exec(`CREATE TABLE IF NOT EXISTS users (
		id SERIAL PRIMARY KEY,
		email VARCHAR(255) UNIQUE NOT NULL,
		name VARCHAR(255) NOT NULL,
		created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
	)`)

	// gRPC server
	lis, err := net.Listen("tcp", ":50051")
	if err != nil {
		log.Fatal(err)
	}
	s := grpc.NewServer()
	pb.RegisterUserServiceServer(s, &server{db: db})
	go func() {
		log.Println("gRPC server listening on :50051")
		if err := s.Serve(lis); err != nil {
			log.Fatal(err)
		}
	}()

	// REST server
	r := gin.Default()
	r.GET("/health", func(c *gin.Context) {
		c.JSON(200, gin.H{"status": "healthy"})
	})
	r.Run(":8081")
}
EOF

# Create Dockerfile
cat > Dockerfile << 'EOF'
FROM golang:1.21-alpine AS builder

WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download

COPY . .
RUN go build -o user-service .

FROM alpine:latest
RUN apk --no-cache add ca-certificates
WORKDIR /root/
COPY --from=builder /app/user-service .
EXPOSE 50051 8081
CMD ["./user-service"]
EOF
```

### Step 11: Create Remaining Services

Repeat Step 10 for:
- `product-service` (gRPC + REST, MongoDB)
- `order-service` (gRPC + Kafka, PostgreSQL)
- `payment-service` (gRPC, PostgreSQL)
- `inventory-service` (gRPC + Kafka, DynamoDB)
- `notification-service` (Kafka consumer, MongoDB)
- `search-service` (REST, Elasticsearch)

---

## Containerization

### Step 12: Create Docker Compose for Local Development

```bash
cd ../.. # Back to project root

cat > docker-compose.yml << 'EOF'
version: '3.8'

services:
  postgres:
    image: postgres:14
    environment:
      POSTGRES_DB: movister
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data

  mongodb:
    image: mongo:6
    ports:
      - "27017:27017"
    volumes:
      - mongo_data:/data/db

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

  zookeeper:
    image: confluentinc/cp-zookeeper:latest
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000

  kafka:
    image: confluentinc/cp-kafka:latest
    depends_on:
      - zookeeper
    ports:
      - "9092:9092"
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1

  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.8.0
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
    ports:
      - "9200:9200"

  api-gateway:
    build: ./services/api-gateway
    ports:
      - "8080:8080"
      - "8081:8081"
    depends_on:
      - postgres
      - mongodb
      - redis

  user-service:
    build: ./services/user-service
    ports:
      - "50051:50051"
      - "8081:8081"
    environment:
      DATABASE_URL: postgres://postgres:postgres@postgres:5432/movister?sslmode=disable
    depends_on:
      - postgres

volumes:
  postgres_data:
  mongo_data:
EOF
```

---

## Kubernetes Deployment

### Step 13: Create Kubernetes Base Manifests

```bash
# Create base deployment template
cat > kubernetes/base/deployment.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: SERVICE_NAME
spec:
  replicas: 2
  selector:
    matchLabels:
      app: SERVICE_NAME
  template:
    metadata:
      labels:
        app: SERVICE_NAME
    spec:
      containers:
      - name: SERVICE_NAME
        image: IMAGE_NAME:latest
        ports:
        - containerPort: 8080
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: url
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
---
apiVersion: v1
kind: Service
metadata:
  name: SERVICE_NAME
spec:
  selector:
    app: SERVICE_NAME
  ports:
  - port: 80
    targetPort: 8080
  type: ClusterIP
EOF

# Create namespace
cat > kubernetes/base/namespace.yaml << 'EOF'
apiVersion: v1
kind: Namespace
metadata:
  name: movister
EOF
```

### Step 14: Create Helm Chart Template

```bash
cd kubernetes/helm-charts/service-template

# Initialize Helm chart
helm create service-template
cd service-template

# Customize values.yaml and templates
```

### Step 15: Deploy to Kubernetes

```bash
# Create namespace
kubectl create namespace movister

# Apply base configurations
kubectl apply -f kubernetes/base/namespace.yaml

# Deploy services (example for API Gateway)
kubectl apply -f kubernetes/overlays/dev/api-gateway.yaml

# Verify deployments
kubectl get pods -n movister
kubectl get services -n movister
```

---

## CI/CD Pipeline

### Step 16: Create Reusable Workflows

Reusable workflows promote DRY principles by centralizing common CI/CD patterns. This makes it easier to maintain consistency across all services.

```bash
mkdir -p .github/workflows/reusable

# Create reusable workflow for Node.js testing
cat > .github/workflows/reusable/test-nodejs.yml << 'EOF'
name: Test Node.js Service

on:
  workflow_call:
    inputs:
      service_path:
        required: true
        type: string
      service_name:
        required: true
        type: string
      node_version:
        required: false
        type: string
        default: '18'
    secrets:
      NPM_TOKEN:
        required: false

jobs:
  test:
    name: Test ${{ inputs.service_name }}
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ inputs.node_version }}
          cache: 'npm'

      - name: Install dependencies
        working-directory: ${{ inputs.service_path }}
        run: npm ci

      - name: Run tests
        working-directory: ${{ inputs.service_path }}
        run: npm test
EOF

# Create reusable workflow for Go testing
cat > .github/workflows/reusable/test-go.yml << 'EOF'
name: Test Go Service

on:
  workflow_call:
    inputs:
      service_path:
        required: true
        type: string
      service_name:
        required: true
        type: string
      go_version:
        required: false
        type: string
        default: '1.21'

jobs:
  test:
    name: Test ${{ inputs.service_name }}
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Go
        uses: actions/setup-go@v4
        with:
          go-version: ${{ inputs.go_version }}

      - name: Run tests
        working-directory: ${{ inputs.service_path }}
        run: go test -v ./...
EOF

# Create reusable workflow for Docker builds
cat > .github/workflows/reusable/build-docker.yml << 'EOF'
name: Build and Push Docker Image

on:
  workflow_call:
    inputs:
      service_path:
        required: true
        type: string
      service_name:
        required: true
        type: string
    secrets:
      DOCKER_USERNAME:
        required: true
      DOCKER_PASSWORD:
        required: true

jobs:
  build:
    name: Build ${{ inputs.service_name }}
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Login to Docker Registry
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}

      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: ${{ inputs.service_path }}
          push: true
          tags: ${{ secrets.DOCKER_USERNAME }}/${{ inputs.service_name }}:latest
EOF

# Create reusable workflow for Kubernetes deployment
cat > .github/workflows/reusable/deploy-kubernetes.yml << 'EOF'
name: Deploy to Kubernetes

on:
  workflow_call:
    inputs:
      environment:
        required: true
        type: string
      service_name:
        required: true
        type: string
      namespace:
        required: false
        type: string
        default: 'movister'
    secrets:
      AWS_ACCESS_KEY_ID:
        required: true
      AWS_SECRET_ACCESS_KEY:
        required: true

jobs:
  deploy:
    name: Deploy ${{ inputs.service_name }}
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: us-east-1

      - name: Update kubeconfig
        run: |
          aws eks update-kubeconfig --name movister-${{ inputs.environment }} --region us-east-1

      - name: Deploy to Kubernetes
        run: |
          kubectl apply -f kubernetes/overlays/${{ inputs.environment }}/${{ inputs.service_name }}.yaml
          kubectl rollout status deployment/${{ inputs.service_name }} -n ${{ inputs.namespace }}
EOF

# Create reusable workflow for security scanning
cat > .github/workflows/reusable/security-scan.yml << 'EOF'
name: Security Scan

on:
  workflow_call:
    inputs:
      service_path:
        required: true
        type: string
      service_name:
        required: true
        type: string
    secrets:
      GITHUB_TOKEN:
        required: true

jobs:
  security-scan:
    name: Security Scan ${{ inputs.service_name }}
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: 'fs'
          scan-ref: ${{ inputs.service_path }}
          format: 'sarif'
          output: 'trivy-results.sarif'
EOF
```

### Step 17: Create Main CI/CD Workflows Using Reusable Workflows

```bash
# Create main CI workflow that uses reusable workflows
cat > .github/workflows/ci.yml << 'EOF'
name: CI Pipeline

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  # Test API Gateway using reusable workflow
  test-api-gateway:
    uses: ./.github/workflows/reusable/test-nodejs.yml
    with:
      service_path: 'services/api-gateway'
      service_name: 'api-gateway'
      node_version: '18'

  # Test User Service using reusable workflow
  test-user-service:
    uses: ./.github/workflows/reusable/test-go.yml
    with:
      service_path: 'services/user-service'
      service_name: 'user-service'
      go_version: '1.21'

  # Build Docker images using reusable workflow
  build-images:
    needs: [test-api-gateway, test-user-service]
    strategy:
      matrix:
        service:
          - name: api-gateway
            path: services/api-gateway
          - name: user-service
            path: services/user-service
    uses: ./.github/workflows/reusable/build-docker.yml
    with:
      service_path: ${{ matrix.service.path }}
      service_name: ${{ matrix.service.name }}
    secrets:
      DOCKER_USERNAME: ${{ secrets.DOCKER_USERNAME }}
      DOCKER_PASSWORD: ${{ secrets.DOCKER_PASSWORD }}

  # Security scan using reusable workflow
  security-scan:
    uses: ./.github/workflows/reusable/security-scan.yml
    with:
      service_path: '.'
      service_name: 'movister-parallel-porter'
    secrets:
      GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
EOF

# Create main CD workflow that uses reusable workflows
cat > .github/workflows/cd.yml << 'EOF'
name: CD Pipeline

on:
  push:
    branches: [ main ]

jobs:
  deploy-dev:
    uses: ./.github/workflows/reusable/deploy-kubernetes.yml
    with:
      environment: dev
      service_name: 'api-gateway'
      namespace: movister
    secrets:
      AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
      AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
EOF
```

### Benefits of Reusable Workflows

1. **DRY Principle**: Write once, use everywhere
2. **Consistency**: All services use the same testing and deployment processes
3. **Maintainability**: Update workflows in one place
4. **Scalability**: Easy to add new services without duplicating workflow code
5. **Version Control**: All workflow changes tracked in git

See `.github/workflows/reusable/README.md` for detailed documentation on all reusable workflows.
```

---

## Monitoring & Observability

### Step 17: Set Up Prometheus and Grafana

```bash
# Add Prometheus Helm repo
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Install Prometheus
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace

# Access Grafana
kubectl port-forward -n monitoring svc/prometheus-grafana 3000:80
# Login: admin / prom-operator
```

### Step 18: Configure DataDog (Optional)

```bash
# Install DataDog agent
helm repo add datadog https://helm.datadoghq.com
helm install datadog-agent datadog/datadog \
  --set datadog.apiKey=$DD_API_KEY \
  --set datadog.site=datadoghq.com
```

---

## Security Configuration

### Step 19: Set Up AWS Cognito

```bash
# Create Cognito User Pool via AWS CLI or Terraform
aws cognito-idp create-user-pool \
  --pool-name movister-user-pool \
  --auto-verified-attributes email

# Create User Pool Client
aws cognito-idp create-user-pool-client \
  --user-pool-id <pool-id> \
  --client-name movister-client
```

### Step 20: Create Kubernetes Secrets

```bash
# Create database credentials secret
kubectl create secret generic db-credentials \
  --from-literal=url=postgres://user:pass@host:5432/db \
  --namespace movister

# Create API keys secret
kubectl create secret generic api-keys \
  --from-literal=cognito-client-id=<client-id> \
  --from-literal=cognito-client-secret=<client-secret> \
  --namespace movister
```

---

## Testing & Validation

### Step 21: Create Health Check Scripts

```bash
cat > scripts/health-check.sh << 'EOF'
#!/bin/bash

SERVICES=(
  "api-gateway:8080"
  "user-service:8081"
  "product-service:8082"
)

for service in "${SERVICES[@]}"; do
  IFS=':' read -r name port <<< "$service"
  if curl -f http://localhost:$port/health > /dev/null 2>&1; then
    echo "✅ $name is healthy"
  else
    echo "❌ $name is unhealthy"
    exit 1
  fi
done

echo "All services are healthy!"
EOF

chmod +x scripts/health-check.sh
```

### Step 22: Run End-to-End Tests

```bash
# Test API Gateway
curl http://localhost:8080/health

# Test User Service gRPC (using grpcurl)
grpcurl -plaintext localhost:50051 user.UserService/GetUser

# Test database connectivity
kubectl exec -it <pod-name> -n movister -- psql $DATABASE_URL -c "SELECT 1"
```

---

## Final Steps

### Step 23: Production Deployment Checklist

- [ ] All services tested and passing
- [ ] Infrastructure provisioned in production environment
- [ ] Database backups configured
- [ ] Monitoring dashboards set up
- [ ] Security groups and network ACLs configured
- [ ] SSL/TLS certificates installed
- [ ] CI/CD pipeline tested
- [ ] Disaster recovery plan documented
- [ ] Load testing completed
- [ ] Documentation updated

### Step 24: Go Live

```bash
# Deploy to production
cd terraform/environments/production
terraform apply

# Deploy services
kubectl apply -f kubernetes/overlays/production/

# Monitor deployment
kubectl get pods -n movister -w
kubectl logs -f deployment/api-gateway -n movister
```

---

## Troubleshooting

### Common Issues

1. **EKS Cluster Not Accessible**
   ```bash
   aws eks update-kubeconfig --name movister-dev --region us-east-1
   kubectl get nodes
   ```

2. **Database Connection Issues**
   - Check security groups
   - Verify database endpoint
   - Test connectivity from within cluster

3. **Service Not Starting**
   ```bash
   kubectl describe pod <pod-name> -n movister
   kubectl logs <pod-name> -n movister
   ```

4. **Image Pull Errors**
   - Verify Docker registry credentials
   - Check image pull secrets in Kubernetes

---

## Next Steps

- Set up auto-scaling policies
- Configure blue-green deployments
- Implement canary releases
- Set up alerting rules
- Create runbooks for common operations
- Document API specifications
- Set up API versioning strategy

---

## Resources

- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [Terraform AWS Provider](https://registry.terraform.io/providers/hashicorp/aws/latest/docs)
- [gRPC Documentation](https://grpc.io/docs/)
- [Helm Charts](https://helm.sh/docs/)

---

**Congratulations!** You've successfully built the Movister Parallel Porter microservices architecture. 🎉


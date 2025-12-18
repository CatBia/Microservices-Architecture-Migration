# Microservices Architecture Migration

## Project Overview

This project documents an strategic migration from a monolithic application to a microservices architecture for a high-traffic e-commerce platform. The migration improved system scalability, deployment velocity, and team autonomy while maintaining zero downtime.


## Key Achievements

- ✅ Reduced deployment time from weeks to hours
- ✅ Improved system availability from 99.5% to 99.95%
- ✅ Enabled independent team deployments with zero coordination overhead
- ✅ Reduced infrastructure costs by 30% through efficient resource utilization

## Architecture Overview

### Service Communication
- **gRPC**: Inter-service communication for performance-critical paths
- **RESTful APIs**: External-facing services
- **GraphQL (Apollo Server)**: Flexible data querying
- **Kafka**: Event-driven architecture for asynchronous communication

### Data Layer
- **PostgreSQL (RDS Multi-AZ)**: Transactional data
- **MongoDB**: Document storage
- **Redis**: Caching and session management
- **DynamoDB**: High-throughput use cases

### Infrastructure
- **Containerization**: Docker containers for consistent runtime environments
- **Orchestration**: Kubernetes (EKS) with Helm charts
- **Infrastructure as Code**: Terraform modules for VPC, security groups, network ACLs, and service mesh
- **Cloud Platform**: AWS EKS clusters with multi-AZ deployment

### CI/CD Pipeline
- **GitLab CI**: Automated testing and container builds
- **GitHub Actions**: CI/CD workflows
- **Spinnaker**: Advanced deployment strategies (blue-green, canary)
- **Jenkins**: Legacy system integration

### Observability
- **DataDog**: Distributed tracing, metrics, and APM
- **Grafana**: Real-time monitoring dashboards
- **CloudWatch**: AWS-native metrics and logs
- **New Relic**: Application performance monitoring

### Security
- **AWS Cognito**: Service-to-service authentication
- **Azure AD**: Enterprise SSO via SAML/OAuth
- **Network Security**: Security groups and VPN Gateway

## Project Structure

```
.
├── README.md
├── terraform/                 # Infrastructure as Code
│   ├── modules/
│   │   ├── eks/
│   │   ├── vpc/
│   │   ├── rds/
│   │   └── security/
│   ├── environments/
│   │   ├── dev/
│   │   ├── staging/
│   │   └── production/
│   └── main.tf
├── kubernetes/                # Kubernetes manifests
│   ├── base/                  # Base configurations
│   ├── overlays/              # Environment-specific overlays
│   │   ├── dev/
│   │   ├── staging/
│   │   └── production/
│   └── helm-charts/           # Helm chart templates
│       ├── service-template/
│       └── infrastructure/
├── services/                  # Microservices
│   ├── api-gateway/           # API Gateway service
│   ├── user-service/          # User management service
│   ├── product-service/       # Product catalog service
│   ├── order-service/         # Order processing service
│   ├── payment-service/       # Payment processing service
│   ├── inventory-service/     # Inventory management service
│   ├── notification-service/  # Notification service
│   └── search-service/        # Search service
├── shared/                    # Shared libraries and utilities
│   ├── proto/                 # gRPC proto definitions
│   ├── libs/                  # Shared libraries
│   └── config/                # Shared configuration
├── scripts/                   # Deployment and utility scripts
│   ├── deploy.sh
│   ├── migrate.sh
│   └── health-check.sh
├── docs/                      # Documentation
│   ├── architecture/
│   ├── deployment/
│   └── api/
├── .github/                   # GitHub Actions workflows
│   └── workflows/
│       ├── ci.yml
│       ├── cd.yml
│       └── reusable/          # Reusable workflows
│           ├── test-nodejs.yml
│           ├── test-go.yml
│           ├── build-docker.yml
│           ├── deploy-kubernetes.yml
│           ├── security-scan.yml
│           └── README.md
├── .gitlab-ci.yml            # GitLab CI configuration
├── docker-compose.yml        # Local development environment
└── Makefile                  # Common commands
```

## Getting Started

> 📖 **New to this project?** Check out the comprehensive [Step-by-Step Build Guide](docs/BUILD_GUIDE.md) to build Movister Parallel Porter from scratch.
> 
> 🗺️ **Visual Guide:** View the [Build Guide Flowchart](docs/build-flowchart.html) for an interactive mind map of all build steps.

### Prerequisites

- Docker and Docker Compose
- Kubernetes CLI (kubectl)
- Helm 3.x
- Terraform >= 1.0
- AWS CLI configured with appropriate credentials
- Access to AWS EKS cluster

### Local Development

1. **Clone the repository**
   ```bash
   git clone <repository-url>
   cd Microservices-Architecture-Migration
   ```

2. **Start local services**
   ```bash
   docker-compose up -d
   ```

3. **Run services locally**
   ```bash
   make dev
   ```

### Deployment

#### Development Environment
```bash
cd terraform/environments/dev
terraform init
terraform plan
terraform apply

cd ../../kubernetes/overlays/dev
kubectl apply -k .
```

#### Production Deployment
```bash
# Using Spinnaker pipeline
# Or via CI/CD pipeline triggered by git push
```

## Service Details

### API Gateway
- **Technology**: GraphQL (Apollo Server) + REST
- **Port**: 8080
- **Responsibilities**: Request routing, authentication, rate limiting

### User Service
- **Technology**: gRPC + REST
- **Database**: PostgreSQL
- **Port**: 50051 (gRPC), 8081 (REST)
- **Responsibilities**: User management, authentication, profiles

### Product Service
- **Technology**: gRPC + REST
- **Database**: MongoDB
- **Port**: 50052 (gRPC), 8082 (REST)
- **Responsibilities**: Product catalog, search, recommendations

### Order Service
- **Technology**: gRPC + Kafka
- **Database**: PostgreSQL
- **Port**: 50053 (gRPC), 8083 (REST)
- **Responsibilities**: Order processing, order history

### Payment Service
- **Technology**: gRPC
- **Database**: PostgreSQL
- **Port**: 50054 (gRPC)
- **Responsibilities**: Payment processing, transaction management

### Inventory Service
- **Technology**: gRPC + Kafka
- **Database**: DynamoDB
- **Port**: 50055 (gRPC)
- **Responsibilities**: Stock management, inventory updates

### Notification Service
- **Technology**: Kafka consumer
- **Database**: MongoDB
- **Port**: 8084 (REST)
- **Responsibilities**: Email, SMS, push notifications

### Search Service
- **Technology**: REST
- **Database**: Elasticsearch
- **Port**: 8085 (REST)
- **Responsibilities**: Product search, filtering

## Monitoring and Observability

### Accessing Dashboards
- **Grafana**: `https://grafana.<domain>`
- **DataDog**: Access via DataDog console
- **CloudWatch**: AWS Console → CloudWatch

### Key Metrics
- Request latency (p50, p95, p99)
- Error rates
- Throughput (requests per second)
- Resource utilization (CPU, memory)
- Database connection pool usage

## Security

### Authentication Flow
1. External requests authenticate via Azure AD (SAML/OAuth)
2. Service-to-service communication uses AWS Cognito tokens
3. All traffic encrypted in transit (TLS)

### Network Security
- Security groups restrict traffic between services
- VPN Gateway for secure administrative access
- Network ACLs for additional layer of protection

## CI/CD Pipeline

### Reusable Workflows

This project uses **GitHub Actions Reusable Workflows** to promote DRY (Don't Repeat Yourself) principles and maintain consistency across all services.

**Available Reusable Workflows:**
- `test-nodejs.yml` - Test Node.js services with linting and coverage
- `test-go.yml` - Test Go services with linting and coverage
- `build-docker.yml` - Build and push Docker images
- `deploy-kubernetes.yml` - Deploy services to Kubernetes (EKS)
- `security-scan.yml` - Security scanning with Trivy and CodeQL

**Benefits:**
- ✅ Write once, use everywhere
- ✅ Consistent testing and deployment across all services
- ✅ Easy to maintain and update
- ✅ Scalable - add new services without duplicating code

See [`.github/workflows/reusable/README.md`](.github/workflows/reusable/README.md) for detailed documentation.

### GitLab CI
- Automated testing on merge requests
- Container image builds
- Deployment to staging environment

### GitHub Actions
- Automated testing using reusable workflows
- Security scanning
- Deployment workflows with reusable patterns

### Spinnaker
- Blue-green deployments
- Canary releases
- Automated rollback on failure

## Contributing

1. Create a feature branch from `main`
2. Make your changes
3. Ensure all tests pass
4. Submit a pull request

## License

[Specify License]

## GitHub Pages

A visual presentation of this project is available on GitHub Pages:

🌐 **View the project site**: `https://[your-username].github.io/Microservices-Architecture-Migration/`

The GitHub Pages site includes:
- Interactive overview of the migration project
- Visual presentation of achievements and metrics
- Technology stack breakdown
- Service architecture details

The site is automatically deployed via GitHub Actions when changes are pushed to the `main` branch.

## Contact

[Contact Information]


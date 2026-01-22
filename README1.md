# Cloud-Native Multi-Region DevOps Delivery Pipeline
## Complete AWS Infrastructure Documentation

[![AWS](https://img.shields.io/badge/AWS-Cloud-orange?style=flat&logo=amazon-aws)](https://aws.amazon.com/)
[![Kubernetes](https://img.shields.io/badge/Kubernetes-EKS-326CE5?style=flat&logo=kubernetes)](https://kubernetes.io/)
[![Terraform](https://img.shields.io/badge/IaC-Terraform-7B42BC?style=flat&logo=terraform)](https://www.terraform.io/)
[![CloudFormation](https://img.shields.io/badge/IaC-CloudFormation-FF9900?style=flat&logo=amazon-aws)](https://aws.amazon.com/cloudformation/)

---

## Table of Contents
- [Architecture Overview](#architecture-overview)
- [Infrastructure Components](#infrastructure-components)
- [Traffic Flow](#traffic-flow)
- [Component Placement](#component-placement)
- [Multi-Region Setup](#multi-region-setup)
- [Technology Stack](#technology-stack)
- [Network Architecture](#network-architecture)
- [Security and Compliance](#security-and-compliance)
- [CI/CD Pipeline](#cicd-pipeline)
- [Interview Key Points](#interview-key-points)

---

## Architecture Overview

This is a **multi-region, highly available, cloud-native 3-tier application** deployed on AWS with automatic failover capabilities. The infrastructure is provisioned using **both CloudFormation (Region 1)** and **Terraform (Region 2)** to demonstrate proficiency with multiple IaC tools.

### High-Level Architecture
- **Application**: Student-Teacher Portal (React frontend + Node.js backend)
- **Database**: Amazon RDS MySQL (Multi-AZ)
- **Container Orchestration**: Amazon EKS
- **Regions**: US-East-1 (CloudFormation) + US-West-1 (Terraform)
- **Failover**: Route 53 DNS-based automatic failover

---

## Infrastructure Components

### 1. DNS and Global Services

| Service | Purpose | Details |
|---------|---------|---------|
| Route 53 | DNS and Failover Routing | Domain: studentteacher.threetierashutoshproject1197.xyz |
| | | Primary: Region 1 ALB, Secondary: Region 2 ALB |
| | | Health checks monitor ALB availability |

### 2. Networking Layer

| Component | CIDR/Details | Availability |
|-----------|--------------|--------------|
| VPC | 10.0.0.0/16 | Single VPC per region |
| Public Subnets | 2 subnets (10.0.0.0/20, 10.0.48.0/20) | 2 AZs |
| Private Subnets | 4 subnets (10.0.16.0/20, 10.0.32.0/20, 10.0.64.0/20, 10.0.80.0/20) | 2 AZs |
| Internet Gateway | 1 per VPC | VPC-attached |
| NAT Gateway | 1 in Public Subnet AZ1 | With Elastic IP |
| Route Tables | 2 (Public + Private) | VPC-wide |

### 3. Compute Layer (EKS)

| Component | Specification | Location |
|-----------|---------------|----------|
| EKS Cluster | Name: MyEKSCluster | Control Plane: AWS-managed (outside VPC) |
| | Kubernetes Version: Latest stable | API Server accessible via ENIs in VPC |
| Worker Nodes | Instance Type: t3.medium | Private Subnets (AZ1 and AZ2) |
| | Auto Scaling: Min 1, Max 3, Desired 2 | |
| Pod Networking | AWS VPC CNI | Pods get IPs from VPC CIDR |
| IAM Roles | EKSClusterRole, EKSNodeGroupRole | Managed AWS policies attached |

### 4. Application Layer

| Component | Details | Container Port | Service Port |
|-----------|---------|----------------|--------------|
| Frontend Pods | React Application | 3000 | 80 (ClusterIP) |
| | Image: ECR frontend:v1.{build} | | |
| Backend Pods | Node.js/Express API | 5000 | 80 (ClusterIP) |
| | Image: ECR backend:v1.{build} | | |
| | Environment: DB credentials from K8s Secrets | | |
| Namespace | mern | | |
| Ingress | AWS ALB Ingress Controller | | |
| | Path routing: / and /backend | | |

### 5. Database Layer

| Component | Specification | Location |
|-----------|---------------|----------|
| RDS MySQL | Engine: MySQL 8.0.36 | Private Subnets (AZ1 and AZ2) |
| | Instance Class: db.t3.small | DB Subnet Group spans 2 AZs |
| | Storage: 20 GB | |
| | Multi-AZ: Enabled | Primary + Standby |
| Security Group | Port 3306 from VPC | |

### 6. Load Balancing

| Component | Details | Location |
|-----------|---------|----------|
| Application Load Balancer | Scheme: Internet-facing | Public Subnets (both AZs) |
| | Listener: Port 80 (HTTP) | |
| | Target Type: IP | Direct to pod IPs |
| | Management: K8s AWS Load Balancer Controller | |

### 7. Container Registry

| Component | Details |
|-----------|---------|
| ECR Repositories | frontend, backend |
| | Region-specific repositories |

---

## Traffic Flow

## Complete Request Journey (Internet to Database)

### Step 1: User Request
- **Location**: Internet
- **Action**: User makes HTTP request to `studentteacher.threetierashutoshproject1197.xyz`

### Step 2: DNS Resolution (Route 53)
- **Location**: Global AWS Service
- **Actions**:
  - Receives DNS query
  - Checks health of Primary ALB (Region 1)
  - If healthy: Returns Region 1 ALB DNS
  - If unhealthy: Returns Region 2 ALB DNS (Failover)

### Step 3: VPC Entry (Internet Gateway)
- **Location**: VPC Boundary
- **Action**: Traffic enters VPC from public internet

### Step 4: Load Balancer (Public Subnets)
- **Location**: Public Subnets (AZ1 and AZ2)
- **Actions**:
  - ALB receives HTTP request on Port 80
  - **Path-based routing**:
    - **Path `/`**: Routes to Frontend Service → Serves React static files
    - **Path `/backend`**: Routes to Backend Service → Processes API requests
  - Directly targets Pod IPs (Target Type: IP)
  - **Note**: Both paths are exposed through ALB because the React app runs in the user's browser and makes API calls directly to `/backend`

### Step 5: Kubernetes Service Layer
- **Location**: Internal Cluster Networking
- **Actions**:
  - Frontend Service (ClusterIP:80) routes to Frontend Pods
  - Backend Service (ClusterIP:80) routes to Backend Pods
  - Services provide load balancing across pods

### Step 6: Application Pods (Private Subnets)
- **Location**: EKS Worker Nodes in Private Subnets
- **Actions**:
  
  **Frontend Pod** (Container Port 3000):
  - Acts as a **static file server** (nginx/serve)
  - Serves HTML, CSS, and JavaScript files for React app
  - **Does NOT execute React code or make API calls**
  - **Does NOT communicate with Backend Pods**
  
  **Backend Pod** (Container Port 5000):
  - Node.js API server handles requests from user's browser
  - Reads DB credentials from Kubernetes Secrets
  - Connects to RDS MySQL database
  - Processes API logic and returns JSON responses

### Step 7: Database (Private Subnets)
- **Location**: Private Subnets (AZ1 and AZ2)
- **Actions**:
  - Primary RDS instance in AZ1 receives query
  - Standby instance in AZ2 (synchronous replication for high availability)
  - Returns query results to Backend Pods

### Step 8: Response Flows

**Two Distinct Traffic Patterns:**

#### Pattern A: Initial Page Load (One-time)

**REQUEST:**
User Browser → Route 53 → IGW → ALB (/) → Frontend Pod

**RESPONSE:**
Frontend Pod → ALB → IGW → User Browser (Downloads React app files)

#### Pattern B: API Requests/Responses (Ongoing)

**REQUEST:**
User Browser (React app running) → Route 53 → IGW → ALB (/backend) → Backend Pod → RDS

**RESPONSE:**
RDS → Backend Pod → ALB → IGW → User Browser

**Key Point**: After the initial page load, **Frontend Pod is no longer involved**. All subsequent API communication happens between the user's browser and Backend Pod through ALB.

---

## Outbound Traffic Flow

Pods in Private Subnets access the internet via:

**Flow:** Pod (Private Subnet) → NAT Gateway (Public Subnet) → Internet Gateway → Internet

**Use cases**:
- Pulling Docker images from Amazon ECR
- npm/apt package downloads
- External API calls
- Software updates

---

## Architecture Summary

| Component | Role | Communicates With |
|-----------|------|-------------------|
| **Frontend Pod** | Static file server (nginx/serve) | Only ALB (serves files on request) |
| **Backend Pod** | API server (Node.js) | ALB (receives requests) + RDS (queries database) |
| **User Browser** | Runs React app, makes API calls | ALB (for both static files and API endpoints) |
| **ALB** | Routes traffic based on path | Frontend Pod (`/`) and Backend Pod (`/backend`) |
| **Frontend ↔ Backend Pods** | **NO DIRECT COMMUNICATION** | N/A |

---

## Interview Talking Points

> "The architecture uses a **client-side React pattern** where the frontend pod serves static files once, and then all API communication happens directly between the user's browser and the backend through the Application Load Balancer. The frontend and backend pods don't communicate with each other—they're completely decoupled. The ALB exposes both the frontend at path `/` and the backend API at path `/backend` because the React application runs client-side in the user's browser, not server-side in the pod."


### EKS Control Plane Communication

Worker Nodes (Private Subnets) communicate with EKS Control Plane (AWS-Managed, Outside VPC) and kubectl commands

Communication via VPC ENIs created by AWS, Secure TLS connections, and IAM authentication

---

## Component Placement

### Public Subnets (Internet-routable)

**Deployed Here:**
- Application Load Balancer (spans both AZs)
- NAT Gateway (in AZ1 only)

**NOT Here:**
- EKS Worker Nodes
- Application Pods
- RDS Database

Route Table: 0.0.0.0/0 to Internet Gateway, 10.0.0.0/16 to Local VPC

### Private Subnets (No direct internet access)

**Deployed Here:**
- Private Subnet 1 (AZ1 and AZ2): EKS Worker Nodes
- Private Subnet 2 (AZ1 and AZ2): RDS MySQL instances
- All application pods run on worker nodes

**NOT Here:**
- No resources with public IP addresses
- No direct internet-facing components

Route Table: 0.0.0.0/0 to NAT Gateway, 10.0.0.0/16 to Local VPC

### Outside VPC (AWS-Managed)

- EKS Control Plane: Kubernetes API server, etcd, scheduler, controller manager
- ECR: Container image registry (regional service)
- Route 53: Global DNS service

---

## Multi-Region Setup

### Region 1: US-East-1 (CloudFormation)

| Aspect | Details |
|--------|---------|
| IaC Tool | AWS CloudFormation |
| VPC CIDR | 10.0.0.0/16 |
| RDS Endpoint | mysqldatabase.cvuswm0m6bsc.us-east-1.rds.amazonaws.com |
| ECR Repos | 314146295673.dkr.ecr.us-east-1.amazonaws.com/frontend |
| | 314146295673.dkr.ecr.us-east-1.amazonaws.com/backend |
| Route 53 Record | Primary (with health check) |

### Region 2: US-West-1 (Terraform)

| Aspect | Details |
|--------|---------|
| IaC Tool | Terraform |
| Terraform State | S3: threetier-tf-bucket-ashutosh |
| State File Key | us-east-1/terraform.tfstate |
| VPC CIDR | 10.0.0.0/16 |
| RDS Endpoint | mysqldatabase.c72gscqo01wl.us-west-1.rds.amazonaws.com |
| ECR Repos | 314146295673.dkr.ecr.us-west-1.amazonaws.com/frontend |
| | 314146295673.dkr.ecr.us-west-1.amazonaws.com/backend |
| Route 53 Record | Secondary (failover target) |

### Infrastructure Parity

Both regions have identical infrastructure with only these differences:
- IaC tool used (CloudFormation vs Terraform)
- Region-specific AWS resource identifiers
- Terraform state management in Region 2

---

## Technology Stack

### Infrastructure as Code
- CloudFormation (Region 1): YAML templates
- Terraform (Region 2): HCL configuration with S3 backend

### Container and Orchestration
- Docker: Application containerization
- Amazon EKS: Kubernetes (managed control plane)
- AWS VPC CNI: Pod networking
- AWS Load Balancer Controller: Ingress management

### Application Stack
- Frontend: React.js (port 3000)
- Backend: Node.js with Express.js (port 5000)
- Database: MySQL 8.0.36 (Amazon RDS)

### CI/CD and Quality
- AWS CodePipeline: Orchestration
- AWS CodeBuild: Build execution
- SonarQube: Static code analysis
- Trivy: Container vulnerability scanning
- GitHub: Source code repository

### Monitoring
- Prometheus: Metrics collection
- Grafana: Dashboard visualization
- Helm: Kubernetes package management

---

## Network Architecture

### VPC Design

VPC: 10.0.0.0/16

Public Subnets (2):
- AZ1: 10.0.0.0/20 (4,096 IPs)
- AZ2: 10.0.48.0/20 (4,096 IPs)

Private Subnets (4):
- AZ1 - Subnet 1: 10.0.16.0/20 [EKS Nodes]
- AZ1 - Subnet 2: 10.0.32.0/20 [RDS Primary]
- AZ2 - Subnet 1: 10.0.64.0/20 [EKS Nodes]
- AZ2 - Subnet 2: 10.0.80.0/20 [RDS Standby]

### Security Groups

**RDS Security Group**  
Ingress: Port 3306, Protocol TCP, Source 0.0.0.0/0  
Egress: Default (all traffic allowed)

**EKS Node Security Group (Auto-created)**  
Ingress: From EKS Control Plane, From other worker nodes, From ALB  
Egress: To Anywhere (for internet access via NAT)

### IAM Roles

**EKS Cluster Role**  
Managed Policies: AmazonEKSClusterPolicy, AmazonEKSVPCResourceController  
Trust: eks.amazonaws.com

**EKS Node Group Role**  
Managed Policies: AmazonEKSWorkerNodePolicy, AmazonEKS_CNI_Policy, AmazonEC2ContainerRegistryReadOnly, AmazonSSMManagedInstanceCore  
Custom Inline Policy: EC2 management, EKS nodegroup operations  
Trust: ec2.amazonaws.com

---

## Security and Compliance

### Network Security

**Implemented:**
- Private subnets for all application workloads
- Internet access only via NAT Gateway
- Security groups with least privilege
- Multi-AZ deployment for high availability

### Application Security

**Implemented:**
- SonarQube Scanning: Code quality and security vulnerabilities
- Trivy Scanning: Container image CVE detection
- Kubernetes Secrets: Base64-encoded credentials
- ECR Image Scanning: Automated vulnerability detection
- IAM Roles: No hardcoded AWS credentials

### Database Security

**Implemented:**
- RDS in private subnets
- Security group restricts access to port 3306
- Multi-AZ for automatic failover
- Automated backups

**Production Recommendations:**
- Enable RDS encryption at rest
- Use AWS Secrets Manager
- Implement TLS/SSL for ALB
- Restrict RDS security group to VPC CIDR
- Enable VPC Flow Logs
- Implement AWS WAF on ALB

---

## CI/CD Pipeline

### Pipeline Architecture

**Pipeline 1: Infrastructure Deployment**  
Source (GitHub) to Build (CodeBuild) to Deploy (CloudFormation/Terraform)

**Pipeline 2: Application Deployment**

Install Phase:
- Node.js 18
- SonarQube Scanner
- Trivy

Pre-Build Phase:
- Set image tag (v1.BUILD_NUMBER)
- Login to ECR and Docker Hub
- npm install
- Run SonarQube scan

Build Phase:
- Build Docker images
- Tag images for ECR
- Scan backend with Trivy
- Scan frontend with Trivy

Post-Build Phase:
- Push images to ECR
- Generate Kubernetes manifests with dynamic tags
- Inject region-specific DB secrets
- Output artifacts

Deploy:
- kubectl apply to EKS cluster

### Build Artifacts

Generated Kubernetes manifests:
- frontend-deployment.yaml
- frontend-service.yaml
- backend-deployment.yaml
- backend-service.yaml
- database-namespace.yaml
- database-secrets.yaml
- ingress.yaml
- trivy-backend-report.json
- trivy-frontend-report.json

### Environment Variables

ACCOUNT_ID, REGION, FRONTEND_REPO, BACKEND_REPO, IMAGE_TAG (dynamic), HOST_US_EAST_1, HOST_US_WEST_1, DB_US_EAST_1, DB_US_WEST_1 (all base64 encoded)

---

## Interview Key Points

### Architecture Summary (30-Second Pitch)

I built a multi-region, highly available 3-tier application on AWS with automatic failover. The infrastructure uses CloudFormation in one region and Terraform in another, deploying a React frontend and Node.js backend on Amazon EKS, with a Multi-AZ RDS MySQL database. Route 53 handles DNS-based failover between regions, and the entire deployment is automated via CodePipeline with SonarQube and Trivy security scanning.

### Component Placement (Be Precise)

**Public Subnets:** Application Load Balancer ONLY, NAT Gateway ONLY

**Private Subnets:** EKS Worker Nodes, All application pods, RDS MySQL (both primary and standby)

**Outside VPC:** EKS Control Plane, ECR, Route 53

### Traffic Flow (Memorize)

User Request to Route 53 to Internet Gateway to ALB (Public Subnets) to Pod IPs (Private Subnets via VPC CNI) to Backend Pod to RDS MySQL (Private Subnets)

Outbound Traffic: Pods to NAT Gateway to Internet Gateway to Internet

### Security Measures

1. Network Isolation: All app components in private subnets
2. Code Quality: SonarQube static analysis
3. Vulnerability Scanning: Trivy scans all container images
4. Secrets Management: Kubernetes Secrets (base64)
5. Multi-AZ: RDS automatically fails over
6. IAM Roles: No hardcoded credentials

### Why Multi-Region

- High Availability: Automatic failover if Region 1 fails
- Disaster Recovery: Complete infrastructure replica
- IaC Proficiency: Demonstrates both CloudFormation and Terraform skills
- DNS-based Failover: Route 53 health checks detect outages

### Common Interview Questions

**Q: Why are worker nodes in private subnets?**  
A: For security best practices. They do not need public IPs. They access the internet via NAT Gateway for pulling images and packages, and the ALB routes inbound traffic to them.

**Q: Why is the ALB in public subnets?**  
A: The ALB needs to be internet-facing to receive user requests. It then forwards traffic to private pod IPs using target type IP.

**Q: Why both CloudFormation and Terraform?**  
A: To demonstrate proficiency with both tools and to show that the same architecture can be implemented with different IaC solutions. It also proves infrastructure consistency across regions.

**Q: How does failover work?**  
A: Route 53 performs health checks on the primary ALB. If it fails, DNS automatically resolves to the secondary region ALB within 60 seconds.

**Q: Why Multi-AZ for RDS?**  
A: For database high availability. If the primary instance fails, RDS automatically fails over to the standby in a different AZ with minimal downtime.

### Key Metrics

- Regions: 2 (US-East-1, US-West-1)
- Availability Zones: 2 per region (4 total)
- VPC CIDR: 10.0.0.0/16 (65,536 IPs)
- Subnets: 6 per region (2 public, 4 private)
- EKS Nodes: 2-3 t3.medium instances
- Container Images: 2 (frontend, backend)
- Database: MySQL 8.0.36, db.t3.small, 20GB

### AWS Services Used

Compute: EKS, EC2, ECR  
Networking: VPC, Subnets, IGW, NAT Gateway, Route Tables, EIP, Route 53, ALB  
Database: RDS MySQL (Multi-AZ)  
Security: Security Groups, IAM Roles/Policies, Kubernetes Secrets  
DevOps: CodePipeline, CodeBuild, GitHub  
Storage: S3 (Terraform state), EBS (RDS)  
IaC: CloudFormation, Terraform

---

## Repository Links

1. Overview Repo: [Cloud-Native-DevOps-Delivery-Pipeline](https://github.com/ashutoshsanghi3/Cloud-Native-DevOps-Delivery-Pipeline)
2. CloudFormation Repo: [Student-Teacher-Portal](https://github.com/ashutoshsanghi3/Student-Teacher-Portal)
3. Terraform Repo: [TF-EKS-RDS](https://github.com/ashutoshsanghi3/TF-EKS-RDS)

---

**Last Updated**: January 22, 2026  
**Author**: Ashutosh Sanghi  
**Architecture**: Multi-Region Cloud-Native 3-Tier Application with Kubernetes and Auto-Failover

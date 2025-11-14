# Cloud-Native DevOps Delivery Pipeline 
## AWS-CFT-TF-EKS-RDS

## GitHub Repositories

- **CloudFormation (Region 1):**  
  🔗 https://github.com/ashutoshsanghi3/Student-Teacher-Portal  
  *(Contains: CloudFormation.yaml, buildspec.yaml)*

- **Terraform (Region 2):**  
  🔗 https://github.com/ashutoshsanghi3/TF-EKS-RDS  
  *(Contains: All .tf files for VPC, EKS, RDS, etc.)*

---

## Overview

This project demonstrates a multi-region infrastructure and deployment setup on AWS using two Infrastructure as Code (IaC) tools:

- **AWS CloudFormation (CFT)** for infrastructure provisioning in **one AWS region**.
- **Terraform (TF)** for infrastructure provisioning in **another AWS region**.

In both regions, the project provisions an **Amazon EKS cluster** to deploy a **3-tier application** backed by an **Amazon RDS database**. The infrastructure is customized per region using the respective IaC tool.

The application deployment pipelines integrate:

- **Route 53 failover routing** to automatically redirect traffic between regions for high availability.
- **SonarQube** for static code analysis and quality gates during the CI/CD process.
- **Trivy** for container image vulnerability scanning of the frontend and backend before deployment.
- **Prometheus and Grafana** for Kubernetes-native monitoring and dashboard visualization.

This setup ensures a secure, resilient, observable, and automated multi-region application architecture.

---

## Architecture

### Region 1: CloudFormation-based Infra & CodePipelines

- **Infrastructure provisioning**: AWS CloudFormation templates create:
  - Custom VPC and networking resources
  - Amazon EKS cluster
  - RDS database instance
- **AWS CodePipeline workflows**:
  - **Pipeline 1**: Creates/updates the CloudFormation stack
  - **Pipeline 2:**
    - Runs SonarQube scans on the application code
    - Builds Docker images for frontend and backend
    - Performs Trivy vulnerability scans on both images
    - Pushes the scanned images to Amazon ECR
    - Deploys application manifests to the EKS cluster

---

### Region 2: Terraform-based Infra & CodePipelines

- **Infrastructure provisioning**: Terraform code provisions similar resources as in Region 1:
  - Custom VPC and networking
  - EKS cluster
  - RDS database
- **AWS CodePipeline workflows**:
  - Pipeline to apply Terraform configurations
  - Application pipeline:
    - Runs SonarQube scans
    - Builds Docker images
    - Performs Trivy vulnerability scans on both frontend and backend images
    - Pushes secure images to ECR
    - Deploys application to EKS

---

## 3-Tier Application

The deployed application consists of:

- **Frontend**: React application hosted on EKS
- **Backend**: Node.js/Express API server running in EKS
- **Database**: Amazon RDS MySQL instance

---

## SonarQube Integration

- SonarQube scans are integrated into the CodePipelines to ensure code quality.
- The scanner runs during the build phase and blocks deployment on failing quality gates.

---

## Trivy Integration

- The pipeline now includes **Trivy vulnerability scans** for both **backend and frontend Docker images**.
- Trivy helps detect **OS-level CVEs**, language-specific vulnerabilities, and misconfigurations.
- The build fails if critical or high vulnerabilities are found in any image.
- This enforces secure image delivery before deployment.

---

## Prometheus & Grafana Monitoring (New)

- **Prometheus** is deployed using Helm charts to scrape Kubernetes and application metrics:
  - kubelet, kube-state-metrics, cAdvisor
  - Optional exporters for MySQL and Trivy scan metrics

- **Grafana** is configured with pre-built dashboards showing:
  - Pod CPU & memory usage
  - HTTP request rate and error codes
  - Database performance metrics
  - Kubernetes cluster health

- This stack provides a **real-time observability layer** for application performance and infrastructure health.

---

## Setup Instructions

### Prerequisites

- AWS CLI configured with appropriate permissions
- AWS CodePipeline and CodeBuild configured in both regions
- Docker installed for building images locally if needed
- Access to SonarQube server with authentication token
- Git installed

---

### Deploying in Region 1 (CloudFormation)

1. Run the **CloudFormation stack creation CodePipeline** to provision infrastructure.
2. Run the **application CodePipeline** which:
   - Executes SonarQube scans
   - Builds Docker images and scan the images (Trivy)
   - Pushes images to ECR
   - Deploys app manifests to EKS

---

### Deploying in Region 2 (Terraform)

1. Apply Terraform configurations either manually or via the Terraform CodePipeline.
2. Run the **application CodePipeline** (similar to Region 1) for SonarQube scan, Trivy scan, build, and deploy.

---

### Accessing the Application

- The Kubernetes manifests include an **Ingress** resource.
- This Ingress provisions an AWS Load Balancer that exposes the frontend and backend services externally.
- You can access the application via the Load Balancer's DNS name provided by the Ingress.

---

## Route 53 Failover Routing Configuration

To provide high availability and automatic failover between the two regions (CloudFormation-based and Terraform-based), this project uses **Route 53 failover routing** for the application domain:

- **Domain:** `studentteacher.threetierashutoshproject1197.xyz`
- **Failover Configuration:**
  - Two Route 53 A (or Alias) records are created under this domain, each pointing to the AWS Load Balancers provisioned by the two infrastructure setups:
    - **Primary Record:** Points to the CloudFormation provisioned Load Balancer in Region 1.
    - **Secondary (Failover) Record:** Points to the Terraform provisioned Load Balancer in Region 2.
  - Health checks are configured on the primary load balancer's endpoint to detect availability.
  - If the health check fails, Route 53 automatically fails over DNS resolution to the secondary load balancer.


### Implementation Notes
- Ensure both load balancers have DNS names registered as Alias targets in Route 53.
- Create appropriate health checks targeting application endpoints (e.g., `/health` path on the frontend or backend).
- Update DNS TTL (time-to-live) to a low value (e.g., 60 seconds) for quicker failover response.
- Configure your SSL certificates accordingly for the domain on both load balancers if using HTTPS.

## Notes
- Ensure CodePipeline environments have required IAM roles and permissions.
- Kubernetes manifests use image tags generated dynamically during builds.
- Secrets like DB credentials are stored securely using Kubernetes Secrets.
- Adjust region-specific parameters and secrets in the pipeline environment variables.

---

## Benefits

- Users are automatically routed to the healthy application endpoint without manual intervention.
- Provides **multi-region resilience and high availability**.
- Enables **security-first CI/CD** with SonarQube and Trivy scans.
- Ensures **automated failover** using Route 53 health checks.
- Adds **real-time observability** using Prometheus and Grafana.
- Delivers **fully automated pipelines** for infrastructure provisioning and application deployment.


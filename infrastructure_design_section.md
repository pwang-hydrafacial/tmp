# Infrastructure Foundation

## Core Infrastructure Architecture

The Analytical Batch Processing platform is built on two foundational AWS-managed clusters that provide the underlying compute, networking, and operational infrastructure to support secure multi-tenant analytical workloads.

### Amazon EKS Cluster Configuration

**Cluster Architecture**:
* **High Availability**: Multi-AZ deployment across three availability zones with managed control plane for 99.95% SLA
* **Node Groups**: Mixed instance types optimized for compute-intensive analytical workloads (C5, M5, R5 families)
* **Network Security**: Private subnets with no internet access, security groups restricting inter-pod communication
* **Internal Connectivity**: Access to Fannie Mae's internal Posit Package Manager for R/Python dependency resolution and business data sources for analytical workloads
* **Storage**: Ephemeral storage for temporary job artifacts, with outputs directed to S3 buckets

**Platform Services**:
* **Karpenter Node Provisioner**: Automatic node lifecycle management with spot instance integration for cost optimization
* **Secrets Store CSI Driver**: Integration with AWS Secrets Manager for secure credential mounting
* **Cluster Autoscaler**: Backup scaling mechanism for node group management

**Observability Stack**:
* **CloudWatch Container Insights**: Cluster-level metrics and logging aggregation
* **AWS X-Ray**: Distributed tracing for job execution workflows
* **Prometheus/Grafana**: Custom metrics collection for analytical job performance monitoring

### Amazon MWAA Environment Configuration

**Environment Specifications**:
* **Airflow Version**: Apache Airflow 2.x with custom plugins for EKS integration
* **Environment Size**: Auto-scaling between small and large based on DAG execution demand
* **Network Configuration**: Private web server with VPC endpoint connectivity to EKS cluster
* **Storage**: S3-backed DAG repository with versioning enabled for rollback capabilities and GitLab CI/CD access for automated DAG deployment

**Operational Features**:
* **DAG Synchronization**: Automated sync from S3 bucket with configurable refresh intervals
* **Log Management**: CloudWatch Logs integration with structured logging for audit trails
* **Monitoring**: CloudWatch metrics for task success rates, execution duration, and queue depth
* **Plugin Management**: Custom plugins for EksPodOperator enhancements and credential management

### Network Architecture

**Simplified Connectivity**:
* **Shared VPC**: Both EKS and MWAA clusters are deployed within the same VPC for simplified network connectivity and reduced operational complexity

### Infrastructure Management

**Deployment and Configuration**:
* **Infrastructure as Code**: Terraform modules for repeatable cluster provisioning

This infrastructure foundation provides enterprise-grade reliability, security, and cost-effectiveness while abstracting operational complexity from analytical users and maintaining strict compliance with organizational standards.
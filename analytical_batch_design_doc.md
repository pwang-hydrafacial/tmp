# Analytical Batch Processing Design Document

## Purpose

The existing EC2-hosted R platform on Amazon Linux 2 will reach end-of-life in Q2 2025. While Ubuntu offers vendor support, it is not permitted under Fannie Mae's EC2 OS standards, requiring us to transition to a new architecture.

We have selected the **Posit Suite** and a custom-built **Analytical Batch Processing** capability as our new solution. This design document addresses the **Analytical Batch Processing component**.

## Overview

The Analytical Batch Processing component runs batch analytics workloads (R and Python) on Amazon EKS. Users push code to GitLab, supply runtime parameters, and launch jobs under their assigned **Non-User Unique ID (NUID)** within the security context section of the EKS POD YAML file.

The **Analytical Batch Processing component** consists of an EKS cluster and an MWAA cluster that together provide a secure, multi-tenant batch-processing environment. Jobs are scheduled by AWS MWAA, code and configuration are stored in GitLab, and dynamic auto-scaling is handled by Karpenter. Each workload runs as an R or Python container under the caller's Non-Human Unique ID (NUID) with least-privilege isolation and tightly controlled data access.

## Key Design Highlights

### Trigger & Orchestration
* MWAA launches Kubernetes jobs on EKS
* Jobs are defined and versioned in a GitLab repository
* GitLab CI/CD handles build and deployment processes

### Multi-Tenancy & Identity Isolation
* Every job runs with `runAsUser=<NUID>` and `runAsGroup=<NUID>`
* Jobs have no access to shared platform resources

### Security & Secrets Management
* An init container retrieves CyberArk mTLS client certificates from AWS Secrets Manager via the Kubernetes Secrets Store CSI driver
* Only the init container holds the IRSA service account token; it is not mounted into the main container, preventing credential leakage
* Runtime containers authenticate to CyberArk using the mounted mTLS certificates to retrieve additional secrets

### Data Access
* Data source access uses federated IAM role assumption mapped to the NUID
* Database credentials are scoped to each individual NUID

### Resource Management
* Compute capacity scales automatically with Karpenter based on job demand

## Application Design


### key steps

#### Validate NUID Ownership

The "Validate" stage is the first step in the CI/CD pipeline that ensures security, compliance, and code quality before proceeding to DAG building and deployment. This comprehensive validation process performs multiple checks to prevent unauthorized access and ensure job integrity.

##### Validation Components

The validation stage executes three critical validation scripts in sequence:

**1. Configuration Validation (`validate-config.sh`)**
* **Schema Validation**: Validates `batch-config.yaml` and `runtime.yaml` against predefined JSON schemas
* **Parameter Range Checks**: Ensures resource requests/limits are within acceptable bounds
* **Required Field Verification**: Confirms all mandatory configuration parameters are present
* **Format Validation**: Validates cron expressions, S3 paths, email formats, and IAM role ARNs
* **Cross-Reference Checks**: Ensures consistency between related configuration values

**2. NUID Ownership Validation (`validate-nuid-ownership.sh`)**
* **Identity Verification**: Confirms the caller's NUID matches the `runAs.nuid` specified in `runtime.yaml`
* **Asset Authorization**: Validates that the NUID has permissions for the specified `runAs.asset` 
* **Multi-Tenancy Enforcement**: Prevents users from running jobs under unauthorized NUIDs
* **Audit Trail Generation**: Logs validation attempts for security monitoring
* **Role Mapping Verification**: Ensures the specified AWS IAM role is authorized for the NUID

**3. Code Syntax Validation (`validate-code-syntax.sh`)**
* **Language-Specific Parsing**: Validates R or Python code syntax based on `runtime.yaml` language setting
* **Dependency Resolution**: Checks that specified packages in `runtime.yaml` are available and compatible
* **Security Scanning**: Scans code for prohibited functions or security vulnerabilities
* **Best Practice Enforcement**: Validates against coding standards and patterns
* **Import/Library Verification**: Ensures all imported libraries are approved for use

##### Validation Workflow

The validation process follows this sequence:

1. **Pipeline Trigger**: GitLab CI/CD pipeline initiates when code is pushed to the repository
2. **Environment Setup**: Pipeline extracts NUID and asset name from environment variables (`$(NUID)`, `$(ASSET_NAME)`)
3. **Configuration Check**: `validate-config.sh` processes both YAML files and reports any schema violations
4. **Security Verification**: `validate-nuid-ownership.sh` performs identity and authorization checks
5. **Code Quality Check**: `validate-code-syntax.sh` validates the analytical code for syntax and security
6. **Validation Results**: Pipeline proceeds to build stage only if all validations pass; otherwise, it fails with detailed error messages

##### Failure Handling

If any validation fails:
* **Immediate Pipeline Termination**: The pipeline stops and does not proceed to build or deploy stages
* **Detailed Error Reporting**: Specific validation failures are logged with actionable error messages
* **Security Alerts**: NUID ownership violations trigger security team notifications
* **Developer Feedback**: Clear guidance is provided for resolving validation issues

This comprehensive validation ensures that only authorized, properly configured, and syntactically correct batch jobs proceed through the deployment pipeline.

#### Build MWAA DAG

The "Build MWAA DAG" step in the CI/CD pipeline translates declarative configuration files (`batch-config.yaml` and `runtime.yaml`) into executable MWAA DAG Python files. This process occurs after NUID ownership validation and before DAG deployment to the MWAA S3 bucket.

##### Configuration Input Files

The DAG builder consumes two configuration files:

**batch-config.yaml** - Job orchestration parameters:
* **Schedule Definition**: Cron expressions for job timing (`schedule.expression`)
* **Resource Allocation**: CPU, memory, and storage requests/limits for the Kubernetes pod
* **Execution Policies**: Timeout values and retry policies for fault tolerance
* **Output Configuration**: Data format specifications and S3 destination paths
* **Notification Settings**: Email and Slack alert configurations for job status

**runtime.yaml** - Environment and execution parameters:
* **Language Runtime**: Specifies R or Python version and base container image
* **Security Context**: NUID and group mappings for multi-tenant isolation
* **Environment Variables**: Job-specific configuration including secret references
* **AWS Integration**: IAM role ARNs and region settings for cloud resource access
* **Dependencies**: R/Python package specifications and system-level requirements

##### DAG Generation Process

The build process follows these steps:

1. **Configuration Validation**: Parse and validate both YAML files against predefined schemas to ensure all required fields are present and values are within acceptable ranges

2. **Template Selection**: Choose the appropriate DAG template based on the language runtime specified in `runtime.yaml` (R or Python)

3. **Parameter Injection**: Populate the DAG template with configuration values:
   - DAG metadata (name, description, tags) from `batch-config.yaml`
   - Schedule interval from `schedule.expression` 
   - Default arguments including retries, timeout, and email settings
   - Task configuration for the EksPodOperator

4. **Pod Specification Assembly**: Generate the Kubernetes pod specification by merging:
   - Base pod template with security context (`runAs.nuid`, `runAs.group`)
   - Container image and resource limits from both config files
   - Environment variables and secret references
   - Volume mounts for init container certificate retrieval

5. **DAG File Generation**: Output a complete Python DAG file that can be executed by MWAA

##### Configuration Mapping Example

For the customer churn analysis project:

```yaml
# batch-config.yaml → DAG parameters
name: "customer-churn-analysis" → DAG ID
schedule.expression: "0 3 * * *" → schedule_interval='0 3 * * *'
retryPolicy.limit: 2 → default_args['retries']=2
timeout: 3600 → default_args['execution_timeout']=timedelta(seconds=3600)
notifications.email → default_args['email']=['team@example.com']

# runtime.yaml → EksPodOperator configuration
image: "ml-r4.2.1:v1.5" → EksPodOperator.image
runAs.nuid: 10577 → pod_template securityContext.runAsUser
aws.role → serviceAccount annotations in pod template
environment.variables → EksPodOperator.env_vars
```

The generated DAG file is then ready for deployment to the MWAA DAG S3 bucket, where it becomes available for scheduling and execution.

#### Init Container Set Security Credentials

The "Init Container Set Security Credentials" step establishes a secure credential management pattern that provides least-privilege access to secrets while maintaining strict isolation between the credential retrieval process and the main analytical workload. This design prevents credential leakage and ensures multi-tenant security.

##### Security Architecture

The init container pattern implements a two-stage security model:

**Stage 1: Privileged Credential Retrieval (Init Container)**
* **IRSA Token Access**: The init container runs with the Kubernetes service account that has IRSA (IAM Roles for Service Accounts) permissions
* **AWS Secrets Manager Integration**: Uses the Kubernetes Secrets Store CSI driver to retrieve CyberArk mTLS client certificates
* **Credential Isolation**: The IRSA service account token is only accessible to the init container, not the main workload container
* **Certificate Preparation**: Downloads and configures mTLS certificates for CyberArk authentication

**Stage 2: Unprivileged Workload Execution (Main Container)**
* **No Direct AWS Access**: The main container has no access to IRSA tokens or AWS Secrets Manager
* **Certificate-Based Authentication**: Authenticates to CyberArk using the mTLS certificates prepared by the init container
* **NUID Security Context**: Runs with the restricted security context (`runAsUser=<NUID>`, `runAsGroup=<NUID>`)

##### Init Container Workflow

The credential setup process follows this sequence:

1. **Pod Initialization**: Kubernetes creates the pod with both init container and main container specifications
2. **Service Account Binding**: The pod's service account provides IRSA permissions only to the init container
3. **CSI Driver Mount**: The Secrets Store CSI driver mounts AWS Secrets Manager as a volume accessible to the init container
4. **Certificate Retrieval**: Init container downloads CyberArk mTLS client certificates from AWS Secrets Manager
5. **Shared Volume Setup**: Certificates are written to a shared volume accessible by the main container
6. **Permission Configuration**: File permissions are set to allow access by the target NUID
7. **Init Container Completion**: Init container exits successfully, signaling readiness for main container startup
8. **Main Container Launch**: Main container starts with access to prepared certificates but no AWS credentials

##### Volume and Mount Configuration

The credential sharing mechanism uses Kubernetes volumes:

```yaml
# Shared volume for certificate transfer
volumes:
- name: cyberark-certs
  emptyDir: {}
- name: secrets-store
  csi:
    driver: secrets-store.csi.k8s.io
    readOnly: true
    volumeAttributes:
      secretProviderClass: "cyberark-certificates"

# Init container volume mounts
initContainers:
- name: credential-setup
  volumeMounts:
  - name: secrets-store
    mountPath: "/mnt/secrets"
    readOnly: true
  - name: cyberark-certs
    mountPath: "/shared/certs"

# Main container volume mounts  
containers:
- name: analytics-workload
  volumeMounts:
  - name: cyberark-certs
    mountPath: "/opt/certs"
    readOnly: true
```

##### Security Benefits

This init container pattern provides several security advantages:

**Credential Isolation**
* AWS credentials (IRSA token) never accessible to the main workload
* Temporary credential exposure limited to init container lifecycle
* No persistent AWS credentials stored in the main container

**Least Privilege Access**
* Init container has minimal permissions (only certificate retrieval)
* Main container runs with NUID-specific privileges only
* No shared access to platform-level secrets or resources

**Multi-Tenant Security**
* Each job's certificates are scoped to the specific NUID
* No cross-tenant credential access possible
* Audit trail maintained for all certificate retrievals

**Defense in Depth**
* Multiple layers of authentication (IRSA → AWS Secrets Manager → CyberArk → Data Sources)
* Certificate rotation handled transparently by the platform
* Automatic cleanup of temporary credentials after job completion

##### CyberArk Integration

Once the main container has access to the mTLS certificates:

1. **CyberArk Authentication**: Main container establishes secure connection to CyberArk using the mTLS certificates
2. **Secret Retrieval**: Requests database credentials and other secrets scoped to the NUID
3. **Environment Variable Resolution**: Resolves `{{$SECRET:}}` references in `runtime.yaml` environment variables
4. **Data Source Access**: Uses retrieved credentials to access authorized data sources and databases

This design ensures that sensitive credentials are managed centrally through CyberArk while maintaining strict isolation and least-privilege access principles throughout the analytical batch processing workflow.
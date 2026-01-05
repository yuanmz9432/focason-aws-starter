# Focason AWS Starter

![Architecture](architecture.jpg "Architecture")

## Pre-deployment Preparation

### Prerequisites
Before starting the deployment, ensure you have the following:

1. **AWS Account**
   * Create an AWS account if you don't have one
   * Ensure your IAM user has appropriate permissions (AdministratorAccess recommended, or at minimum: ECS, ECR, VPC, IAM, CloudWatch, ALB, RDS, EC2, Route53 permissions)

2. **AWS CLI Installation and Configuration**
   ```bash
   # Install AWS CLI (if not installed)
   # Windows: Download from https://aws.amazon.com/cli/
   # macOS: brew install awscli
   # Linux: sudo apt-get install awscli
   
   # Configure AWS CLI with your credentials
   aws configure
   # Enter your Access Key ID, Secret Access Key, region (e.g., ap-northeast-1), and output format (json)
   ```

3. **Docker Installation**
   * Install Docker Desktop or Docker Engine
   * Verify installation: `docker --version`

4. **Get AWS Account ID and Region**
   ```bash
   # Get your AWS Account ID
   aws sts get-caller-identity --query Account --output text
   
   # Get your configured region
   aws configure get region
   ```
   * Save these values as you'll need them for ECR repository URIs

5. **Domain and Route53 Hosted Zone**
   * Create a hosted zone in Route53
   * Transfer the NS records of your domain to Route53
   * Note: This must be completed before deployment as some services may require domain configuration

## Deployment Steps

> **Important**: Execute the steps in order as they have dependencies. Each step must complete successfully before proceeding to the next.

### Step 1: Create VPC, Subnet, Security Group, ALB, etc.

This step creates the foundational networking infrastructure.

1. Navigate to AWS CloudFormation Console
2. Click "Create stack" → "With new resources (standard)"
3. Upload the template file: `aws-cloudformation/001_vpc.yaml`
4. Configure stack parameters:
   * **ProjectName**: Default is `focason-cloud` (or customize as needed)
   * **VpcCIDR**: Default is `10.0.0.0/16` (or customize)
   * Other subnet CIDRs can be customized if needed
5. Review and create the stack
6. Wait for stack creation to complete (approximately 5-10 minutes)

**Verification**:
```bash
# Verify VPC was created
aws ec2 describe-vpcs --filters "Name=tag:Name,Values=focason-cloud-vpc"

# Verify ALB was created
aws elbv2 describe-load-balancers --query "LoadBalancers[?contains(LoadBalancerName, 'focason')]"
```

**Note**: Save the stack outputs (especially ALB DNS name and ARN) as they may be needed for later steps.

### Step 2: Create EC2 Bastion Host and RDS Database

#### 2.1 Generate Key Pair

In AWS, EC2 KeyPair is a public/private key pair:
* Public key is stored in AWS
* Private key must be saved by yourself (AWS does not store it)
* Typically used to SSH into EC2 instances.

**Option A: Generate locally and import to AWS**
```bash
# Generate key pair locally
ssh-keygen -t rsa -b 4096 -f focason-keypair

# This generates:
# - focason-keypair (private key) - KEEP THIS SECURE!
# - focason-keypair.pub (public key)
```

**Option B: Create via AWS Console**
1. Go to EC2 → Key Pairs → Create key pair
2. Name: `focason-cloud-keypair`
3. Key pair type: RSA
4. Private key file format: `.pem` (for OpenSSH)
5. Download and securely store the private key

**Option C: Create via CloudFormation (if template supports it)**
If your CloudFormation template includes KeyPair resource, you can add the public key:
```yaml
KeyPair:
  Type: AWS::EC2::KeyPair
  Properties:
    KeyName: !Sub "${ProjectName}-keypair"
    PublicKeyMaterial: |
      ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQCuC+Xv0FQ6M0JcKkZzM... your public key ...
```

#### 2.2 Deploy Bastion Host and RDS

1. Navigate to AWS CloudFormation Console
2. Create a new stack using `aws-cloudformation/002_bastion_rds.yaml`
3. Configure stack parameters:
   * **ProjectName**: Must match the value used in Step 1 (default: `focason-cloud`)
   * **DBUsername**: Database master username (default: `admin`)
   * **DBPassword**: Database master password (minimum 8 characters, keep secure!)
   * **DBName**: Initial database name (default: `focason`)
4. Review and create the stack
5. Wait for stack creation to complete (approximately 10-15 minutes for RDS)

**Verification**:
```bash
# Verify RDS instance was created
aws rds describe-db-instances --query "DBInstances[?contains(DBInstanceIdentifier, 'focason')]"

# Verify bastion host was created
aws ec2 describe-instances --filters "Name=tag:Name,Values=*bastion*" "Name=instance-state-name,Values=running"

# Get bastion host public IP
aws ec2 describe-instances --filters "Name=tag:Name,Values=*bastion*" --query "Reservations[*].Instances[*].PublicIpAddress" --output text
```

**Important**: 
* Save the RDS endpoint (e.g., `mysql-rds.focason.com`) - you'll need it for database connections
* Save the bastion host public IP for SSH access
* Ensure the DB password is securely stored

### Step 3: Create ECS Cluster

1. Navigate to AWS CloudFormation Console
2. Create a new stack using `aws-cloudformation/003_cluster.yaml`
3. Configure stack parameters:
   * **ProjectName**: Must match previous steps (default: `focason-cloud`)
4. Review and create the stack
5. Wait for stack creation to complete (approximately 2-3 minutes)

**Verification**:
```bash
# Verify ECS cluster was created
aws ecs describe-clusters --clusters focason-cloud-cluster

# Verify Task Execution Role was created
aws iam get-role --role-name focason-cloud-ecs-execution-role
```

**Note**: This step also creates the ECS Task Execution Role with necessary permissions for ECR, CloudWatch, and Secrets Manager.

### Step 4: Create ECR (Elastic Container Registry) Repositories

Before deploying ECS services, you need to create ECR repositories to store Docker images for your services.

#### 4.1 Get AWS Account ID and Region

If you haven't already, get these values:
```bash
# Get AWS Account ID
export AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

# Get AWS Region
export AWS_REGION=$(aws configure get region)

# Display values
echo "Account ID: $AWS_ACCOUNT_ID"
echo "Region: $AWS_REGION"
```

#### 4.2 Create ECR Repositories

Create ECR repositories for each service:

**Via AWS Console**:
1. Navigate to ECR → Repositories → Create repository
2. Create repositories with the following names:
   * `focason-service-registry`
   * `focason-config-server`
   * `focason-gateway`
   * `focason-platform-service`
   * (Add other service repositories as needed)
3. For each repository:
   * Visibility: Private
   * Tag immutability: Enable (recommended)
   * Scan on push: Enable (recommended for security)

**Via AWS CLI**:
```bash
# Set variables (replace with your values)
export AWS_ACCOUNT_ID="your-account-id"
export AWS_REGION="ap-northeast-1"

# Create repositories
aws ecr create-repository --repository-name focason-service-registry --region $AWS_REGION
aws ecr create-repository --repository-name focason-config-server --region $AWS_REGION
aws ecr create-repository --repository-name focason-gateway --region $AWS_REGION
aws ecr create-repository --repository-name focason-platform-service --region $AWS_REGION
```

#### 4.3 Note Repository URIs

After creating repositories, note down the repository URIs. Format:
```
<account-id>.dkr.ecr.<region>.amazonaws.com/<repository-name>
```

Example:
```
442426889980.dkr.ecr.ap-northeast-1.amazonaws.com/focason-service-registry
```

**Verification**:
```bash
# List all repositories
aws ecr describe-repositories --query "repositories[*].repositoryUri" --output table
```

#### 4.4 Build and Push Docker Images to ECR

For each service, build and push the Docker image:

```bash
# Set variables
export AWS_ACCOUNT_ID="your-account-id"
export AWS_REGION="ap-northeast-1"
export REPOSITORY_NAME="focason-service-registry"  # Change for each service

# Authenticate Docker to ECR
aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com

# Navigate to your service directory (adjust path as needed)
cd /path/to/your/service

# Build Docker image
docker build -t $REPOSITORY_NAME:latest .

# Tag the image
docker tag $REPOSITORY_NAME:latest $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/$REPOSITORY_NAME:latest

# Push to ECR
docker push $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/$REPOSITORY_NAME:latest
```

**Important**: 
* Update the image URIs in your task definition files (`aws-task-definitions/*.json`) to match your ECR repository URIs
* Ensure all required services have images pushed to ECR before proceeding to Step 5

**Verification**:
```bash
# List images in a repository
aws ecr list-images --repository-name focason-service-registry --region $AWS_REGION
```

### Step 5: Register ECS Task Definitions

Before creating ECS services, you need to register the task definitions.

1. **Update Task Definition Files**
   * Open each task definition file in `aws-task-definitions/`
   * Update the `image` field with your ECR repository URI (from Step 4.3)
   * Update any environment variables, secrets, or configuration as needed
   * Example:
     ```json
     {
       "family": "focason-service-registry-task",
       "containerDefinitions": [{
         "image": "YOUR_ACCOUNT_ID.dkr.ecr.YOUR_REGION.amazonaws.com/focason-service-registry:latest",
         ...
       }]
     }
     ```

2. **Register Task Definitions**
   ```bash
   # Register each task definition
   aws ecs register-task-definition --cli-input-json file://aws-task-definitions/focason-service-registry-task.json
   aws ecs register-task-definition --cli-input-json file://aws-task-definitions/focason-config-server-task.json
   aws ecs register-task-definition --cli-input-json file://aws-task-definitions/focason-gateway-task.json
   aws ecs register-task-definition --cli-input-json file://aws-task-definitions/focason-platform-service-task.json
   ```

**Verification**:
```bash
# List registered task definitions
aws ecs list-task-definitions --family-prefix focason

# View a specific task definition
aws ecs describe-task-definition --task-definition focason-service-registry-task
```

### Step 6: Create ECS Base Services

This step deploys the core microservices:
* Eureka (Service Discovery)
* RabbitMQ (Message Queue)
* Config Server (Configuration Management)
* Gateway (API Gateway)
* Authentication Service

1. Navigate to AWS CloudFormation Console
2. Create a new stack using `aws-cloudformation/004_ecs_base.yaml`
3. Configure stack parameters:
   * **ProjectName**: Must match previous steps (default: `focason-cloud`)
4. Review and create the stack
5. Wait for stack creation to complete (approximately 5-10 minutes)

**Verification**:
```bash
# List ECS services
aws ecs list-services --cluster focason-cloud-cluster

# Check service status
aws ecs describe-services --cluster focason-cloud-cluster --services <service-name>

# View running tasks
aws ecs list-tasks --cluster focason-cloud-cluster
```

**Troubleshooting**:
* If services fail to start, check CloudWatch Logs for errors
* Verify task definitions are registered correctly
* Ensure ECR images are accessible
* Check security group rules allow necessary traffic

### Step 7: Create Frontend Service (NUXT)

1. Navigate to AWS CloudFormation Console
2. Create a new stack using `aws-cloudformation/005_nuxt_static_hosting.yaml`
3. Configure stack parameters as needed
4. Review and create the stack
5. Wait for stack creation to complete

6. **Deploy Frontend Files to S3**
   In your frontend project directory, execute:
   ```bash
   sh deploy-nuxt-to-s3.sh
   ```
   
   **Note**: Ensure the deployment script is configured with the correct S3 bucket name and region.

**Verification**:
```bash
# Verify S3 bucket was created
aws s3 ls | grep focason

# Verify CloudFront distribution (if created)
aws cloudfront list-distributions --query "DistributionList.Items[?contains(Comment, 'focason')]"
```

## Post-deployment Operations

### Database Initialization

1. **Configure Database Connection**
   * Modify the configuration information in `gradle.properties` in the `focason-core` module
   * Update database connection settings:
     * Host: RDS endpoint (from Step 2)
     * Port: 3306
     * Database: focason (or your DB name)
     * Username: admin (or your DB username)
     * Password: (your DB password)

2. **Establish Connection Tunnel Locally**
   * Refer to [Connecting to Database Locally via Bastion Host](#connecting-to-database-locally-via-bastion-host)

3. **Execute Flyway Tasks**
   * Check Flyway execution status:
     ```bash
     ./gradlew flywayInfo
     ```
   * Clean database (if needed):
     ```bash
     ./gradlew flywayClean
     ```
   * Execute pending DDL:
     ```bash
     ./gradlew flywayMigrate
     ```
   * **Quick access via IDE**:
     ```
     Gradle -> focason-cloud -> focason-core -> Tasks -> flyway
     ```

### Page Verification

1. Access the application:
   * Frontend: https://focason.com (or your configured domain)
   * API Gateway: Check ALB DNS name from Step 1

2. Test the application:
   * Register a new user
   * Login with credentials
   * Verify core functionality

3. If accessible normally, deployment is complete! ✅

### Monitoring and Logs

* **CloudWatch Logs**: View application logs
  ```bash
  aws logs tail /ecs/focason-cloud-services --follow
  ```

* **ECS Service Status**: Monitor service health
  ```bash
  aws ecs describe-services --cluster focason-cloud-cluster --services <service-name>
  ```

* **CloudWatch Metrics**: Monitor CPU, memory, and network metrics

## Bastion Host

### Component Updates

Update the EC2 instance software to get the latest bug fixes and security updates:
```bash
# SSH into bastion host
ssh -i /path/to/your-key.pem ec2-user@<bastion-public-ip>

# Update system packages
sudo dnf update -y
```

### Connect to Database

1. **Install MariaDB Client** (if not already installed):
   ```bash
   sudo dnf install mariadb105
   ```

2. **Connect to MySQL DB Instance**:
   ```bash
   mysql -h mysql-rds.focason.com -P 3306 -u admin -p
   # Enter password when prompted
   ```

## Database

### Connecting to Database Locally via Bastion Host

#### Option 1: Using Termius (GUI Tool)

1. **Open Termius**
2. **Create Keychain**
   * Add your private key file (`.pem` or the private key from Step 2.1)
3. **Connect to Host**
   * Hostname(IP): Public IP of the bastion host in AWS
   * User: `ec2-user`
   * Key: The keychain created in the previous step
4. **Create Port Forwarding (Local Port Mapping)**
   * Type: Local
   * Label: transfer 3306 to 13306
   * Local port number: `13306`
   * Bind address: `127.0.0.1`
   * Intermediate host: Bastion host (the connection you created above)
   * Destination address: RDS Endpoint (e.g., `mysql-rds.focason.com`)
   * Destination port number: `3306`
   * Username: `ec2-user`
5. **Connect to DB using Local DB Tool** (e.g., A5M2, DBeaver, MySQL Workbench)
   * Hostname: `localhost`
   * Port: `13306`
   * User ID: `admin` (or your RDS username)
   * Password: (your RDS password)
   * Database: `focason` (or your database name)

#### Option 2: Using SSH Command Line

Execute the following command locally:
```bash
ssh -i /path/to/your-key.pem -L 13306:mysql-rds.focason.com:3306 -N ec2-user@<bastion-host-public-ip>
```

Then connect to the database using:
* Hostname: `localhost`
* Port: `13306`
* Username: `admin`
* Password: (your RDS password)
* Database: `focason`

**Note**: Keep the SSH session open while using the database connection.

## Troubleshooting

### Common Issues

1. **CloudFormation Stack Creation Fails**
   * Check CloudFormation events for specific error messages
   * Verify IAM permissions are sufficient
   * Ensure parameter values are correct
   * Check if resource limits are reached (e.g., VPC limits)

2. **ECS Tasks Fail to Start**
   * Check CloudWatch Logs: `/ecs/focason-cloud-services`
   * Verify task definition image URI is correct
   * Ensure ECR repository exists and image is pushed
   * Check security group rules allow necessary traffic
   * Verify task execution role has correct permissions

3. **Cannot Connect to RDS**
   * Verify security group allows connections from bastion host
   * Check RDS endpoint is correct
   * Ensure database is in "available" state
   * Verify credentials are correct

4. **Services Cannot Communicate**
   * Check security group rules
   * Verify service discovery is configured correctly
   * Check VPC routing tables
   * Ensure services are in correct subnets

5. **Docker Push to ECR Fails**
   * Verify AWS credentials are configured: `aws configure list`
   * Check ECR repository exists
   * Ensure Docker is authenticated: `aws ecr get-login-password --region <region> | docker login ...`
   * Verify image is tagged correctly with full ECR URI

### Useful Commands

```bash
# Check CloudFormation stack status
aws cloudformation describe-stacks --stack-name <stack-name>

# View CloudFormation stack events
aws cloudformation describe-stack-events --stack-name <stack-name> --max-items 10

# Check ECS service status
aws ecs describe-services --cluster focason-cloud-cluster --services <service-name>

# View ECS task logs
aws logs tail /ecs/focason-cloud-services --follow

# Check RDS instance status
aws rds describe-db-instances --db-instance-identifier <instance-id>

# List ECR repositories
aws ecr describe-repositories

# Check security group rules
aws ec2 describe-security-groups --group-ids <sg-id>
```

## Security Best Practices

1. **Key Management**
   * Never commit private keys to version control
   * Use AWS Secrets Manager for sensitive data (passwords, API keys)
   * Rotate credentials regularly

2. **IAM Permissions**
   * Follow principle of least privilege
   * Use IAM roles instead of access keys when possible
   * Regularly audit IAM permissions

3. **Network Security**
   * Keep RDS in private subnets
   * Use security groups to restrict access
   * Enable VPC Flow Logs for monitoring

4. **Container Security**
   * Scan Docker images for vulnerabilities
   * Use specific image tags instead of `latest` in production
   * Keep base images updated

## Cost Optimization Tips

1. **Use Appropriate Instance Sizes**
   * Start with smaller instances (t4g.micro, db.t4g.micro) for development
   * Monitor usage and scale up as needed

2. **Clean Up Unused Resources**
   * Delete unused ECR images
   * Stop/terminate unused EC2 instances
   * Delete unused CloudFormation stacks

3. **Use Reserved Instances** (for production)
   * Consider Reserved Instances for predictable workloads

4. **Monitor Costs**
   * Set up AWS Cost Explorer
   * Configure billing alerts

## Additional Resources

* [AWS CloudFormation Documentation](https://docs.aws.amazon.com/cloudformation/)
* [Amazon ECS Documentation](https://docs.aws.amazon.com/ecs/)
* [Amazon ECR Documentation](https://docs.aws.amazon.com/ecr/)
* [AWS VPC Documentation](https://docs.aws.amazon.com/vpc/)

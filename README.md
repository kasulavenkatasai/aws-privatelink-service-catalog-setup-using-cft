# AWS PrivateLink Service Catalog Setup Using CloudFormation

## 1. Project Overview

This project automates the deployment of AWS PrivateLink infrastructure using AWS CloudFormation and AWS Service Catalog.

It creates two separate VPC environments within the same AWS account:

- **Provider VPC:** Hosts a MariaDB database on an EC2 instance and exposes it privately through a Network Load Balancer (NLB) and VPC Endpoint Service.
- **Consumer VPC:** Hosts a client EC2 instance that connects to the MariaDB database through an Interface VPC Endpoint.

The infrastructure is deployed through a Service Catalog product, allowing users to provision the complete setup without manually creating each resource.

## 2. Architecture

**AWS Account:** Same AWS account  
**AWS Region:** `us-east-1`  
**Connectivity:** AWS PrivateLink  
**Database:** MariaDB

### Network Configuration

| Component | CIDR |
|---|---|
| Provider VPC | `10.50.0.0/16` |
| Consumer VPC | `10.60.0.0/16` |

### Architecture Flow

```text
                  AWS ACCOUNT
                       |
              AWS SERVICE CATALOG
                       |
           PrivateLink Infrastructure Product
                       |
              CloudFormation Stack
                       |
           +-----------+-----------+
           |                       |
      PROVIDER VPC             CONSUMER VPC
      10.50.0.0/16             10.60.0.0/16
           |                       |
       Provider EC2             Consumer EC2
           |                       |
       MariaDB Server          MariaDB Client
           |                       |
       Target Group                |
           |                       |
     Network Load Balancer         |
           |                       |
    VPC Endpoint Service           |
           |                       |
           +---- AWS PrivateLink ---+
                       |
              Interface Endpoint
                       |
               Database Connection
```

## 3. Project Files

| File | Description |
|---|---|
| `.gitignore` | Excludes sensitive and unnecessary files from Git. |
| `privatelink-bootstrap-v3.yaml` | Creates the initial Service Catalog setup, S3 bucket, portfolio, launch role and client group. |
| `privatelink-catalog-access.yaml` | Configures portfolio and product access for the client group. |
| `privatelink-catalog-product-v7.yaml` | Earlier version of the Service Catalog product registration template. |
| `privatelink-catalog-product-v8.yaml` | Registers the latest infrastructure product and its provisioning artifacts. |
| `privatelink-single-stack-v7.yaml` | Earlier version of the complete PrivateLink infrastructure template. |
| `privatelink-single-stack-v8.yaml` | Latest complete infrastructure template for Provider and Consumer environments. |
| `README.md` | Project documentation and deployment instructions. |

## 4. How the Project Works

### Step 1: Bootstrap Setup

The bootstrap CloudFormation template creates the initial resources required for Service Catalog.

It provisions:

- S3 bucket for CloudFormation templates.
- Service Catalog portfolio.
- IAM launch role.
- IAM client group.
- Client group membership.
- Portfolio association.

### Step 2: Upload CloudFormation Templates

Upload the infrastructure template and product registration template to the S3 artifacts bucket.

The templates are stored in S3 so Service Catalog can access them during provisioning.

### Step 3: Register Service Catalog Product

The product registration template creates the Service Catalog product.

The product uses the infrastructure CloudFormation template as its provisioning artifact.

### Step 4: Configure Product Access

The access template associates the product and portfolio with the required IAM group.

This allows authorized users to access and launch the product.

### Step 5: Launch the Product

The user opens AWS Service Catalog and launches the PrivateLink infrastructure product.

The user provides the required parameters, such as:

- EC2 instance type.
- Availability Zone.
- VPC and subnet CIDRs.
- Database name.
- Database username.
- Database password.

CloudFormation provisions the Provider and Consumer infrastructure.

### Step 6: Provider-Side Deployment

The Provider environment creates:

1. Provider VPC.
2. Public subnet and networking resources.
3. Provider EC2 instance.
4. MariaDB server.
5. Network Load Balancer.
6. Target group and listener.
7. VPC Endpoint Service.

### Step 7: Consumer-Side Deployment

The Consumer environment creates:

1. Consumer VPC.
2. Public subnet and networking resources.
3. Consumer EC2 instance.
4. MariaDB client.
5. Interface VPC Endpoint.
6. Security groups and IAM resources.

### Step 8: PrivateLink Connection

The Consumer EC2 instance connects to the Provider MariaDB database through the Interface Endpoint.

The connection follows this path:

```text
Consumer EC2
     |
     v
Interface VPC Endpoint
     |
     v
AWS PrivateLink
     |
     v
Provider VPC Endpoint Service
     |
     v
Network Load Balancer
     |
     v
Provider EC2
     |
     v
MariaDB Database
```

## 5. Prerequisites

Before deploying the project, ensure you have:

- AWS account with appropriate permissions.
- AWS CLI installed.
- AWS CLI configured with credentials.
- CloudFormation access.
- IAM permissions to create required resources.
- Service Catalog access.
- S3 access.
- An internet connection for initial package installation and SSM connectivity.

Check AWS CLI:

```powershell
aws --version
```

Check AWS account:

```powershell
aws sts get-caller-identity
```

Set the AWS region:

```powershell
aws configure set region us-east-1
```

## 6. CloudFormation File Deployment Sequence

Deploy the CloudFormation templates in the following order. Each step depends on resources created in the previous steps.

### Deployment Order

| Order | File | Action |
|---|---|---|
| 1 | `privatelink-bootstrap-v3.yaml` | Deploy using CloudFormation |
| 2 | `privatelink-single-stack-v8.yaml` | Upload to the S3 artifacts bucket |
| 3 | `privatelink-catalog-product-v8.yaml` | Deploy using CloudFormation |
| 4 | `privatelink-catalog-access.yaml` | Deploy using CloudFormation |
| 5 | Service Catalog product | Launch the infrastructure product |

### Step 1: Deploy Bootstrap Template

Deploy `privatelink-bootstrap-v3.yaml` first.

It creates the S3 artifacts bucket, Service Catalog portfolio, IAM launch role and client group.

### Step 2: Upload Infrastructure Template

Upload `privatelink-single-stack-v8.yaml` to the S3 bucket created in Step 1.

This is the main infrastructure template that creates both Provider and Consumer environments.

### Step 3: Register Service Catalog Product

Deploy `privatelink-catalog-product-v8.yaml`.

This registers the PrivateLink infrastructure product and its provisioning artifacts.

### Step 4: Configure Service Catalog Access

Deploy `privatelink-catalog-access.yaml`.

This configures access to the portfolio and product for the required IAM group.

### Step 5: Launch the Service Catalog Product

Open AWS Service Catalog and launch `PrivateLink-Infrastructure-Product`.

CloudFormation automatically deploys the Provider and Consumer infrastructure using the registered infrastructure template.

**Important Notes:**

- The `privatelink-single-stack-v8.yaml` template is uploaded to S3. It is not manually deployed as a separate stack in this workflow.
- The Service Catalog product deploys the infrastructure template when a user launches the product.
- The current v8 product registration references provisioning artifacts v1 through v8. Ensure all referenced artifacts exist at their configured S3 URLs, or modify the registration template to use only v8.
- For another AWS account, update the S3 URLs, portfolio ID, product ID and launch role ARN as applicable.

## 7. Deployment Guide

### Step 1: Clone the Repository

```powershell
git clone https://github.com/kasulavenkatasai/aws-privatelink-service-catalog-setup-using-cft.git
```

Navigate into the project:

```powershell
cd aws-privatelink-service-catalog-setup-using-cft
```

### Step 2: Validate CloudFormation Templates

Validate the bootstrap template:

```powershell
aws cloudformation validate-template `
  --template-body file://privatelink-bootstrap-v3.yaml `
  --region us-east-1
```

Validate the infrastructure template:

```powershell
aws cloudformation validate-template `
  --template-body file://privatelink-single-stack-v8.yaml `
  --region us-east-1
```

### Step 3: Deploy the Bootstrap Stack

```powershell
aws cloudformation deploy `
  --template-file privatelink-bootstrap-v3.yaml `
  --stack-name PrivateLink-Bootstrap-Stack `
  --capabilities CAPABILITY_NAMED_IAM `
  --region us-east-1
```

### Step 4: Upload Templates to S3

Use the bucket created by the bootstrap stack.

Upload the infrastructure template:

```powershell
aws s3 cp privatelink-single-stack-v8.yaml s3://YOUR-S3-BUCKET/
```

Upload the product registration template:

```powershell
aws s3 cp privatelink-catalog-product-v8.yaml s3://YOUR-S3-BUCKET/
```

Replace `YOUR-S3-BUCKET` with the actual bucket name.

### Step 5: Register the Service Catalog Product

```powershell
aws cloudformation deploy `
  --template-file privatelink-catalog-product-v8.yaml `
  --stack-name PrivateLink-Catalog-Product-Stack `
  --capabilities CAPABILITY_NAMED_IAM `
  --region us-east-1
```

### Step 6: Configure Service Catalog Access

```powershell
aws cloudformation deploy `
  --template-file privatelink-catalog-access.yaml `
  --stack-name PrivateLink-Catalog-Access-Stack `
  --capabilities CAPABILITY_NAMED_IAM `
  --region us-east-1
```

Ensure the portfolio ID, product ID and launch role ARN match the resources created in your account.

### Step 7: Launch the Product

1. Open AWS Management Console.
2. Navigate to AWS Service Catalog.
3. Open the portfolio.
4. Select `PrivateLink-Infrastructure-Product`.
5. Choose Launch product.
6. Enter the required parameters.
7. Review the configuration.
8. Launch the product.
9. Monitor the provisioning status.

## 8. Connectivity Testing

After provisioning, connect to the Consumer EC2 instance using AWS Systems Manager Session Manager.

### Test TCP Connectivity

```bash
timeout 5 bash -c '</dev/tcp/ENDPOINT_DNS/3306' \
  && echo "PrivateLink connection successful" \
  || echo "PrivateLink connection failed"
```

Replace `ENDPOINT_DNS` with the Consumer Interface Endpoint DNS name.

### Connect to MariaDB

```bash
mysql -h ENDPOINT_DNS -P 3306 -u DATABASE_USERNAME -p DATABASE_NAME
```

Enter the database password when prompted.

### Verify the Database Connection

Execute:

```sql
SELECT VERSION();
SELECT DATABASE();
SELECT USER();
SHOW TABLES;
```

A successful MariaDB login confirms that the Consumer can access the Provider database through PrivateLink.

## 9. Troubleshooting

| Issue | Possible Solution |
|---|---|
| CloudFormation validation failure | Check YAML syntax and template parameters. |
| Stack creation failure | Review CloudFormation stack events. |
| Endpoint not available | Verify Endpoint Service configuration and permissions. |
| TCP connection failure | Check endpoint security group, NLB and target health. |
| MariaDB connection failure | Verify database status, credentials and security groups. |
| SSM connection failure | Verify instance profile, SSM permissions and outbound connectivity. |
| MariaDB host blocked | On the Provider EC2 instance, run `sudo mariadb-admin flush-hosts`. |

## 10. Security Considerations

- Database traffic is accessed through AWS PrivateLink rather than a publicly exposed database endpoint.
- Database security group rules should only allow traffic from the required NLB security group.
- Use IAM roles instead of hardcoded AWS credentials.
- Protect database passwords and never commit them to GitHub.
- The current template uses EC2 UserData for database initialization. Avoid exposing UserData because it may contain sensitive configuration.
- The bootstrap launch role currently uses broad `PowerUserAccess` permissions. Review and restrict permissions before production use.
- The current architecture uses public subnets and public IPs to support outbound access and SSM connectivity. This does not mean MariaDB is publicly accessible.

## 11. Deployment in Another AWS Account

The current implementation is a same-account, two-VPC demonstration.

If deploying in another AWS account:

- Update the S3 bucket references in the Service Catalog product template.
- Upload the required CloudFormation artifacts to that account's S3 bucket.
- Update portfolio IDs and product IDs.
- Update the launch role ARN.
- Verify IAM permissions and account-specific parameters.
- Verify the selected Availability Zone and CIDR ranges.

The current v8 product registration references historical v1–v8 artifacts. Either upload all referenced artifacts or modify the registration template to use only the required v8 artifact.

## 12. Cleanup

After testing, terminate the Service Catalog provisioned product first.

Then remove the supporting resources in the correct dependency order:

1. Terminate the provisioned product.
2. Delete the Service Catalog product and its supporting associations.
3. Delete the bootstrap stack.
4. Empty the S3 artifacts bucket if required.
5. Verify that no EC2 instances, VPC endpoints, load balancers or other billable resources remain.

Check your AWS Billing dashboard after cleanup.

## 13. Project Outcome

The project demonstrates how AWS CloudFormation and AWS Service Catalog can automate the deployment of AWS PrivateLink infrastructure.

A user can launch one Service Catalog product to provision both Provider and Consumer environments, establish private connectivity, and access a MariaDB database without exposing the database directly to the public internet.

### Technologies Used

AWS CloudFormation, AWS Service Catalog, AWS PrivateLink, VPC, EC2, Network Load Balancer, VPC Endpoint Service, Interface VPC Endpoint, IAM, S3, Systems Manager and MariaDB.

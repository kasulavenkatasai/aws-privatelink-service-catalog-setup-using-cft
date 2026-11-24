# AWS PrivateLink Deployment Using CloudFormation and Service Catalog

## 1. Project Objective

Automate the deployment of Provider and Consumer AWS infrastructure through AWS CloudFormation Templates (CFT) and AWS Service Catalog.

A user launches one reusable Service Catalog product, and CloudFormation provisions the required resources in both VPCs. AWS PrivateLink provides private connectivity from the Consumer environment to a MariaDB database hosted in the Provider environment.

**Architecture type:** Same AWS account, two VPCs.

The Provider and Consumer are separate network environments, not separate AWS accounts.

## 2. Architecture

### Provider Side

- Provider VPC and subnet.
- Internet Gateway and route table for instance package installation and outbound access.
- Security groups.
- Provider EC2 instance running MariaDB.
- IAM role and instance profile for AWS Systems Manager Session Manager.
- Internal Network Load Balancer (NLB), TCP port `3306`.
- Target group pointing to the Provider database EC2 instance.
- VPC Endpoint Service backed by the NLB.
- Endpoint Service permission for the account root principal.

### Consumer Side

- Consumer VPC and subnet.
- Internet Gateway and route table for outbound access.
- Consumer EC2 instance with MariaDB client.
- IAM role and instance profile for Systems Manager Session Manager.
- Interface VPC Endpoint connected to the Provider Endpoint Service.

### Network Configuration

| Component | CIDR |
|---|---|
| Provider VPC | `10.50.0.0/16` |
| Provider Subnet | `10.50.1.0/24` |
| Consumer VPC | `10.60.0.0/16` |
| Consumer Subnet | `10.60.1.0/24` |

### Traffic Flow

```text
Consumer EC2
    |
    | MariaDB TCP/3306
    v
Consumer Interface VPC Endpoint
    |
    | AWS PrivateLink
    v
Provider VPC Endpoint Service
    |
    v
Internal Network Load Balancer
    |
    v
Provider Target Group
    |
    v
Provider EC2 running MariaDB
```

The database connection uses the Interface Endpoint DNS name. MariaDB is not exposed through a public database endpoint.

## 3. Repository Contents

| File | Purpose |
|---|---|
| `privatelink-bootstrap-v3.yaml` | Initial bootstrap stack. Creates the S3 template bucket, Service Catalog portfolio, IAM launch role, IAM client group, group membership and portfolio association. |
| `privatelink-catalog-access.yaml` | Associates the registered Service Catalog product with the portfolio and applies the launch-role constraint. |
| `privatelink-catalog-product-v8.yaml` | Registers the infrastructure template as the `PrivateLink-Infrastructure-Product` Service Catalog product and defines provisioning artifacts v1 through v8. |
| `privatelink-single-stack-v8.yaml` | Main infrastructure template. Creates Provider and Consumer VPC resources, MariaDB EC2, NLB, Endpoint Service, Interface Endpoint, Consumer EC2, SSM roles and outputs. |
| `privatelink-catalog-product-v7.yaml` | Previous product-registration template retained for version history. |
| `privatelink-single-stack-v7.yaml` | Previous infrastructure-template version retained for comparison and version history. |
| `.gitignore` | Excludes local environment files, credentials, private keys, the archive folder and the generated deployment snapshot. |
| `README.md` | Project documentation and deployment instructions. |

`deployed-template.json` and the `archive/` directory are intentionally excluded from the Git repository. They are not required by the documented v8 deployment path.

## 4. How the Deployment Works

1. The bootstrap CloudFormation template creates the S3 bucket and initial Service Catalog and IAM resources.
2. The infrastructure template is uploaded to the S3 bucket.
3. The product-registration CloudFormation template registers the S3-hosted infrastructure template as a Service Catalog product.
4. The catalog-access template associates the product with the portfolio and configures the launch-role constraint.
5. An authorized user opens Service Catalog and launches the product, selecting the v8 provisioning artifact and entering the required parameters.
6. Service Catalog assumes the configured IAM launch role and asks CloudFormation to provision the infrastructure.
7. The main CloudFormation template creates the Provider and Consumer resources and connects them through AWS PrivateLink.
8. The Consumer EC2 instance uses the Interface Endpoint DNS name to connect to MariaDB on the Provider EC2 instance.

## 5. IAM Launch Role and Permissions

The IAM launch role is an important component of AWS Service Catalog provisioning.

It allows AWS Service Catalog to provision infrastructure using a dedicated IAM role instead of relying on the end user's personal permissions for every resource creation operation.

### 5.1 Create the IAM Launch Role

The `privatelink-bootstrap-v3.yaml` template creates the IAM launch role:

```text
PrivateLink-ServiceCatalog-LaunchRole
```

The bootstrap template also creates its associated IAM permissions and outputs the launch role ARN.

### 5.2 Configure Launch Role Permissions

The launch role requires permissions to provision the resources defined in the infrastructure CloudFormation template.

These include permissions for services such as:

- Amazon EC2.
- Amazon VPC.
- Elastic Load Balancing.
- AWS IAM.
- Amazon S3.
- AWS CloudFormation and required supporting operations.

The current bootstrap template attaches the AWS-managed `PowerUserAccess` policy and additional IAM permissions, including permissions required for EC2 instance roles and access to the S3 template bucket.

**Security note:** `PowerUserAccess` is broad. Review and replace it with least-privilege permissions before using this architecture in production.

### 5.3 Associate the Product with the Portfolio

The `privatelink-catalog-access.yaml` template creates the Service Catalog product-to-portfolio association:

```yaml
PrivateLinkPortfolioProductAssociation:
  Type: AWS::ServiceCatalog::PortfolioProductAssociation
  Properties:
    PortfolioId: !Ref PortfolioId
    ProductId: !Ref ProductId
```

This makes the registered product available through the specified portfolio.

### 5.4 Configure the Launch Role Constraint

The same access template creates a launch-role constraint:

```yaml
PrivateLinkLaunchRoleConstraint:
  Type: AWS::ServiceCatalog::LaunchRoleConstraint
  DependsOn:
    - PrivateLinkPortfolioProductAssociation
  Properties:
    PortfolioId: !Ref PortfolioId
    ProductId: !Ref ProductId
    RoleArn: !Ref LaunchRoleArn
    Description: >
      Dedicated launch role for provisioning the AWS PrivateLink
      Provider and Consumer infrastructure.
```

The launch-role constraint tells Service Catalog which IAM role to assume when a user launches the product.

Creating the IAM role alone is not sufficient. The product must also have the launch-role constraint configured.

### 5.5 How the Launch Process Works

```text
Service Catalog User
        |
        v
Launches Product
        |
        v
AWS Service Catalog
        |
        v
Portfolio and Product Association
        |
        v
Launch Role Constraint
        |
        v
IAM Launch Role
        |
        v
AWS CloudFormation
        |
        v
Provider and Consumer Infrastructure
```

### 5.6 Launch Role vs EC2 Instance Role

These are two different IAM roles.

| IAM Role | Purpose |
|---|---|
| Service Catalog Launch Role | Allows Service Catalog to provision infrastructure through CloudFormation. |
| Provider EC2 Instance Role | Provides permissions to the Provider EC2 instance, including Systems Manager access. |
| Consumer EC2 Instance Role | Provides permissions to the Consumer EC2 instance, including Systems Manager access. |

The launch role provisions the infrastructure. The EC2 instance roles provide permissions to the running instances.

## 6. Prerequisites

Before deploying the project, ensure you have:

- An AWS account with permissions to create CloudFormation, Service Catalog, IAM, S3, EC2, VPC, Elastic Load Balancing and VPC Endpoint resources.
- AWS CLI v2 installed and configured.
- PowerShell on Windows or an equivalent terminal.
- A region that supports the required services and instance type. This project uses `us-east-1`.
- An IAM user or role authorized to create the bootstrap and catalog stacks.
- An existing IAM user matching the `ClientUserNames` bootstrap parameter if you intend to use the default value `venkat`.

Check the local tools and AWS identity:

```powershell
aws --version
aws sts get-caller-identity
aws configure get region
```

If no default region is configured, pass `--region us-east-1` to the commands below.

## 7. Get the Project on Your Laptop

Clone the repository:

```powershell
git clone https://github.com/kasulavenkatasai/aws-privatelink-service-catalog-setup-using-cft.git

cd aws-privatelink-service-catalog-setup-using-cft
```

Check the files:

```powershell
Get-ChildItem -Force
```

## 8. Validate CloudFormation Templates

Validate the bootstrap template:

```powershell
aws cloudformation validate-template `
  --template-body file://privatelink-bootstrap-v3.yaml `
  --region us-east-1
```

Validate the catalog-access template:

```powershell
aws cloudformation validate-template `
  --template-body file://privatelink-catalog-access.yaml `
  --region us-east-1
```

Validate the product-registration template:

```powershell
aws cloudformation validate-template `
  --template-body file://privatelink-catalog-product-v8.yaml `
  --region us-east-1
```

Validate the infrastructure template:

```powershell
aws cloudformation validate-template `
  --template-body file://privatelink-single-stack-v8.yaml `
  --region us-east-1
```

## 9. CloudFormation Deployment Sequence

Deploy the templates in the following order because each stage depends on resources created in earlier stages.

| Order | File / Resource | Action |
|---|---|---|
| 1 | `privatelink-bootstrap-v3.yaml` | Deploy bootstrap stack |
| 2 | `privatelink-single-stack-v8.yaml` | Upload infrastructure template to S3 |
| 3 | `privatelink-catalog-product-v8.yaml` | Deploy product-registration stack |
| 4 | `privatelink-catalog-access.yaml` | Associate product and configure launch-role constraint |
| 5 | Service Catalog product | Launch the infrastructure product |

### Step 1: Deploy the Bootstrap Stack

The bootstrap template creates:

- S3 template bucket.
- Service Catalog portfolio.
- IAM launch role.
- IAM client group.
- Client group membership.
- Portfolio principal association.

For the default client username:

```powershell
aws cloudformation create-stack `
  --stack-name PrivateLink-Bootstrap-Stack `
  --template-body file://privatelink-bootstrap-v3.yaml `
  --capabilities CAPABILITY_NAMED_IAM `
  --parameters ParameterKey=ClientUserNames,ParameterValue=venkat `
  --region us-east-1
```

If your IAM username is different, replace `venkat` with an existing IAM username in your account.

Wait for completion:

```powershell
aws cloudformation wait stack-create-complete `
  --stack-name PrivateLink-Bootstrap-Stack `
  --region us-east-1
```

Read the bootstrap outputs:

```powershell
aws cloudformation describe-stacks `
  --stack-name PrivateLink-Bootstrap-Stack `
  --region us-east-1 `
  --query "Stacks[0].Outputs" `
  --output table
```

Record the output values for:

- Template bucket name.
- Portfolio ID.
- Launch-role ARN.
- Client group name and ARN.

### Step 2: Upload the Infrastructure Template to S3

Get the bucket name from the bootstrap stack output.

Set it in PowerShell:

```powershell
$BucketName = "<TEMPLATE-BUCKET-NAME>"
```

Upload the v8 infrastructure template:

```powershell
aws s3 cp .\privatelink-single-stack-v8.yaml "s3://$BucketName/privatelink-single-stack-v8.yaml" --region us-east-1
```

Confirm the object exists:

```powershell
aws s3 ls "s3://$BucketName/" --region us-east-1
```

### Step 3: Configure the Service Catalog Product-Registration Template

**Important:** `privatelink-catalog-product-v8.yaml` contains S3 URLs that point to the original project's S3 bucket.

When deploying in another AWS account, update the v8 `LoadTemplateFromURL` value to the URL of the v8 template uploaded to your own bucket.

Example URL format:

```text
https://<TEMPLATE-BUCKET-NAME>.s3.us-east-1.amazonaws.com/privatelink-single-stack-v8.yaml
```

The template also lists historical artifacts v1 through v7.

If you are creating a fresh environment and do not have those objects, either:

1. Upload the matching historical templates and update their URLs, or
2. Create a simplified product-registration template containing only the v8 artifact.

Do not leave broken S3 URLs in provisioning artifacts you expect users to launch.

Validate the edited registration template:

```powershell
aws cloudformation validate-template `
  --template-body file://privatelink-catalog-product-v8.yaml `
  --region us-east-1
```

### Step 4: Register the Service Catalog Product

After updating the S3 URLs, create the product-registration stack:

```powershell
aws cloudformation create-stack `
  --stack-name PrivateLink-Catalog-Product-Stack `
  --template-body file://privatelink-catalog-product-v8.yaml `
  --capabilities CAPABILITY_NAMED_IAM `
  --region us-east-1
```

If this stack already exists, use `update-stack` instead of `create-stack`:

```powershell
aws cloudformation update-stack `
  --stack-name PrivateLink-Catalog-Product-Stack `
  --template-body file://privatelink-catalog-product-v8.yaml `
  --capabilities CAPABILITY_NAMED_IAM `
  --region us-east-1
```

Check the stack status:

```powershell
aws cloudformation describe-stacks `
  --stack-name PrivateLink-Catalog-Product-Stack `
  --region us-east-1 `
  --query "Stacks[0].[StackName,StackStatus]" `
  --output table
```

Get the product ID:

```powershell
aws servicecatalog search-products `
  --region us-east-1 `
  --query "ProductViewDetails[*].[ProductViewSummary.Name,ProductViewSummary.ProductId]" `
  --output table
```

Record the product ID.

### Step 5: Associate the Product and Configure the Launch Role

The `privatelink-catalog-access.yaml` template requires:

- Portfolio ID.
- Product ID.
- Launch-role ARN.

For the original demonstration account, the template has default values. For another AWS account, provide the IDs and ARN created in that account.

Example command:

```powershell
aws cloudformation create-stack `
  --stack-name PrivateLink-Catalog-Access-Stack `
  --template-body file://privatelink-catalog-access.yaml `
  --parameters `
    ParameterKey=PortfolioId,ParameterValue=<PORTFOLIO-ID> `
    ParameterKey=ProductId,ParameterValue=<PRODUCT-ID> `
    ParameterKey=LaunchRoleArn,ParameterValue=<LAUNCH-ROLE-ARN> `
  --region us-east-1
```

Replace the three placeholders with the actual values from the bootstrap and product-registration stacks.

If the access stack already exists, use `update-stack` instead.

Wait for the stack to complete and verify its status:

```powershell
aws cloudformation describe-stacks `
  --stack-name PrivateLink-Catalog-Access-Stack `
  --region us-east-1 `
  --query "Stacks[0].[StackName,StackStatus]" `
  --output table
```

This stack performs two important operations:

1. Associates the product with the portfolio.
2. Creates the launch-role constraint linking the product to the IAM launch role.

### Step 6: Launch the Service Catalog Product

After the bootstrap, product registration and catalog-access stacks are complete:

1. Open AWS Management Console.
2. Navigate to AWS Service Catalog in `us-east-1`.
3. Open the portfolio.
4. Confirm that `PrivateLink-Infrastructure-Product` is associated with the portfolio.
5. Open the product.
6. Select the v8 provisioning artifact.
7. Choose Launch product.
8. Enter the required parameters.
9. Review the configuration.
10. Launch the product.

The launch parameters include:

| Parameter | Default / Example |
|---|---|
| EC2 instance type | `t3.micro` or `t3.small` |
| Availability Zone | `us-east-1a` |
| Provider VPC CIDR | `10.50.0.0/16` |
| Provider subnet CIDR | `10.50.1.0/24` |
| Consumer VPC CIDR | `10.60.0.0/16` |
| Consumer subnet CIDR | `10.60.1.0/24` |
| Database name | `privatelink_demo` |
| Database username | `privatelink_app` |
| Database password | Enter a new strong password meeting the template constraints |

The database password parameter is marked `NoEcho: true`. Do not place a real password in source control, README files, shell history or chat messages.

**Important:** The `privatelink-single-stack-v8.yaml` file is not manually deployed as a standalone stack in this workflow. Service Catalog uses it to provision the infrastructure when the product is launched.

## 10. Verify the Deployment

Find the provisioned CloudFormation stack created by Service Catalog. The exact stack name is generated by Service Catalog.

List the Service Catalog CloudFormation stacks:

```powershell
aws cloudformation describe-stacks `
  --region us-east-1 `
  --query "Stacks[?contains(StackName, 'SC-')].[StackName,StackStatus]" `
  --output table
```

Inspect the relevant stack resources:

```powershell
aws cloudformation describe-stack-resources `
  --stack-name <SERVICE-CATALOG-CLOUDFORMATION-STACK-NAME> `
  --region us-east-1 `
  --query "StackResources[*].[LogicalResourceId,ResourceType,PhysicalResourceId,ResourceStatus]" `
  --output table
```

Confirm that the expected resources include:

- Provider VPC.
- Consumer VPC.
- Provider database EC2.
- Network Load Balancer.
- VPC Endpoint Service.
- Consumer Interface Endpoint.
- Consumer EC2.

## 11. Test Connectivity from the Consumer EC2 Instance

The infrastructure template creates SSM roles and instance profiles for both EC2 instances.

Retrieve the Consumer EC2 instance ID from the stack resources or outputs.

Start a Session Manager session:

```powershell
aws ssm start-session `
  --target <CONSUMER-INSTANCE-ID> `
  --region us-east-1
```

### 11.1 Resolve the Endpoint DNS Name

On the Consumer EC2 instance, obtain the endpoint DNS name from the CloudFormation output `ConsumerEndpointDnsName` or the VPC endpoint details.

Run:

```bash
nslookup <CONSUMER-ENDPOINT-DNS-NAME>
```

### 11.2 Test TCP Port 3306

```bash
timeout 5 bash -c '</dev/tcp/<CONSUMER-ENDPOINT-DNS-NAME>/3306' \
  && echo "PrivateLink connection successful" \
  || echo "PrivateLink connection failed"
```

### 11.3 Connect to MariaDB

```bash
mariadb -h <CONSUMER-ENDPOINT-DNS-NAME> -P 3306 -u privatelink_app -p privatelink_demo
```

Enter the database password when prompted.

### 11.4 Verify the Database Connection

After login, execute:

```sql
SELECT VERSION();
SELECT DATABASE();
SELECT USER();
SHOW TABLES;
```

A successful MariaDB login from the Consumer EC2 instance through the Interface Endpoint verifies the end-to-end data path.

## 12. Troubleshooting

### Session Manager Cannot Connect

- Confirm the instance is running and appears as a managed node in Systems Manager.
- Confirm the instance profile includes `AmazonSSMManagedInstanceCore`.
- Confirm the instance has outbound connectivity to the Systems Manager endpoints.

### MariaDB Host Is Blocked (ERROR 1129)

On the Provider EC2 instance, run:

```bash
sudo mariadb-admin flush-hosts
```

Then retry the connection from the Consumer.

If the problem returns, investigate repeated failed connections and MariaDB host-cache settings rather than repeatedly retrying passwords.

### TCP 3306 Test Fails

- Confirm the Interface Endpoint state is `Available`.
- Confirm the endpoint DNS name is correct.
- Confirm the Consumer endpoint security group allows TCP 3306 from the Consumer client security group.
- Confirm the Provider database security group allows TCP 3306 from the NLB security group.
- Confirm the NLB target group reports the Provider EC2 target as healthy.
- Confirm MariaDB is running on the Provider EC2 instance.

### Service Catalog Product Launch Fails

- Confirm the product is associated with the portfolio.
- Confirm the launch-role constraint references the correct IAM role.
- Confirm the launch role has the permissions required by the infrastructure template.
- Confirm the IAM role trust policy allows AWS Service Catalog to assume the role.
- Confirm all provisioning-artifact S3 URLs are valid and accessible.
- Confirm the user has access to the Service Catalog portfolio.

## 13. Security and Account-Specific Notes

- This is a demonstration template. Review IAM permissions before using it in a production account.
- The bootstrap launch role attaches the AWS-managed `PowerUserAccess` policy and additional IAM permissions. This is broad access and should be narrowed for production.
- The template creates public subnets and public IP outputs for the EC2 instances to support outbound access and package installation. The database security group permits MariaDB ingress from the Provider NLB security group, not directly from the public internet.
- The Endpoint Service permissions in this template allow the account root principal. Review this setting if adapting the design for cross-account consumers.
- The `DatabasePassword` parameter uses `NoEcho`, but the template also interpolates the password into EC2 UserData for database setup and the Consumer connection test. Treat UserData and instance access as sensitive; do not use this password-handling pattern as a production secret-management design.
- The default CIDRs, Availability Zone, S3 URLs, portfolio ID, product ID and IAM role ARN are environment-specific. Review and update them for your account.
- The project uses one AWS account with two VPCs. A true cross-account design requires additional account-specific Endpoint Service permissions, IAM roles and deployment orchestration.

## 14. Cleanup

First terminate the Service Catalog provisioned product from the Service Catalog console or use the Service Catalog termination workflow.

Confirm the associated provisioned CloudFormation stack and its resources have been deleted.

Only after the product deployment is removed should you delete the catalog access and product-registration stacks, followed by the bootstrap stack.

The bootstrap stack owns the S3 bucket and IAM/portfolio resources, so confirm the bucket is empty and no other product artifacts depend on it before deleting it.

Do not delete shared S3 artifacts or IAM roles while active Service Catalog products or provisioned products still depend on them.

## 15. Project Result

The project demonstrates how:

- AWS CloudFormation defines repeatable infrastructure.
- AWS Service Catalog packages infrastructure as a reusable product.
- IAM launch roles provide controlled provisioning permissions.
- Service Catalog launch-role constraints connect products to the required IAM launch roles.
- AWS PrivateLink provides private connectivity from a Consumer VPC to a MariaDB service in a Provider VPC.

### Technologies Used

AWS CloudFormation, AWS Service Catalog, AWS PrivateLink, VPC, EC2, Network Load Balancer, VPC Endpoint Service, Interface VPC Endpoint, IAM, S3, Systems Manager and MariaDB.

---

**Repository:**  
https://github.com/kasulavenkatasai/aws-privatelink-service-catalog-setup-using-cft

# HPC-Stack: A Slurm-Powered Cloud HPC Solution with GPU Support on Local Machines  

## Introduction  
This project sets up a flexible and cost-effective High-Performance Computing (HPC) cluster using Slurm in Docker containers. The Slurm controller runs on the cloud, while GPU workloads are handled by local on-premises machines. With AWS and Grafana, it ensures efficient resource management and real-time monitoring.  

## Key Objectives  
- **Easy Deployment:** Quickly set up an HPC cluster with containerized Slurm.  
- **Scalability:** Add compute nodes as needed for better performance.  
- **Hybrid Flexibility:** Use the cloud for Slurm management and local machines for GPU processing.  
- **Cost-Effective:** Designed for research and education with minimal costs.  
- **Real-Time Monitoring:** Track resource usage and performance with Grafana.  

## Tech Stack  
- **Orchestration & Monitoring:** Docker, Grafana, Prometheus  
- **HPC Scheduler:** Slurm Workload Manager  
- **Cloud Services:** AWS EC2, VPC  
- **Database:** MySQL (for resource tracking and job management)  
- **Networking:** AWS VPC, Security Groups for secure access  

# Maintaining a structured record of project execution

## 1) AWS VPC and Networking Setup

### Creating a VPC

To establish a secure and isolated network environment, a Virtual Private Cloud (VPC) is created. A VPC allows users to define their own IP address ranges, subnets, and security configurations.

- **VPC Name**: SLURM-VPC
- **CIDR Block**: 10.0.0.0/16
- A VPC ensures a controlled and secure networking environment by isolating resources from other AWS customers.

![VPC](https://github.com/user-attachments/assets/ce4c5d00-f635-4dc2-93df-ee8bfb1e6a60)

### Configuring Subnets

#### Private Subnet (For Controller)

- **Name**: PrivateSubnet
- **CIDR**: 10.0.1.0/24
- **Public IP**: Disabled
- The controller node must remain isolated to prevent unauthorized external access and enhance security.

#### Public Subnet (For Bastion Host)

- **Name**: PublicSubnet
- **CIDR**: 10.0.2.0/24
- **Public IP**: Enabled
- The Bastion Host needs public access to serve as a secure gateway to private resources.

![Subnets](https://github.com/user-attachments/assets/778d27d9-55bb-4b51-ab69-67e34986e13d)

### Attaching an Internet Gateway

An Internet Gateway (IGW) is attached to allow external communication for public instances.

- **Gateway Name**: SLURM-IGW
- **VPC Attachment**: SLURM-VPC
- Required for public-facing instances to communicate with the internet.

![Internet Gateway](https://github.com/user-attachments/assets/8c87dfe9-f7d9-49bc-b710-b30bb8698082)

### Route Table Configuration

#### Public Route Table

- **Name**: Public-RT
- **Association**: PublicSubnet
- **Routes**:
  - `0.0.0.0/0` -> Internet Gateway (SLURM-IGW)
- Ensures that instances in the public subnet can access the internet.

![Public Route Table](https://github.com/user-attachments/assets/ea73f8ae-7d21-4d3f-ba5e-1074edb6ecac)

#### Private Route Table

A NAT Gateway is required for controlled internet access in private subnets.

- **Subnet**: PublicSubnet
- **Elastic IP**: Assigned
- **Routes**:
  - `0.0.0.0/0` -> NAT Gateway (SLURM-NAT)
- Allows private instances to access updates and external resources while preventing unsolicited inbound traffic.

![Private Route Table](https://github.com/user-attachments/assets/c7304190-ffc0-451b-8877-f6cc3b6934c7)

### Why an Elastic IP for NAT Gateway?

- Provides a static public IP
- Ensures stable internet access for private instances
- Prevents IP changes on restart
- Supports failover scenarios
- Ensures that private instances maintain a consistent outbound communication channel.

![Elastic IP for NAT Gateway](https://github.com/user-attachments/assets/160a5a26-93ab-4fdf-9ad4-e8a3509a7128)

![Attach NAT Gateway to Private RT](https://github.com/user-attachments/assets/ca5bbb9a-adad-4175-8e3c-18bdc128b47f)

### Launching AWS Instances

#### Controller Node (Private Subnet)

- **Instance Type**: `t3.medium` or `t3.large`
- **Subnet**: PrivateSubnet
- **Security Group**:
  - SSH access only from Bastion Host (`10.0.0.x/24`)
  - SLURM Ports (`6817-6819`) open for compute nodes
- **Storage**: 20GB SSD
- The controller node manages the SLURM cluster and requires controlled access for cluster operations.

#### Bastion Host (Public Subnet)

- **Instance Type**: `t2.micro`
- **Subnet**: PublicSubnet
- **Security Group**:
  - SSH access limited to a trusted IP range
- **Storage**: 8GB SSD
- The Bastion Host provides a secure method to access private instances, reducing exposure to external threats.

![AWS Instances](https://github.com/user-attachments/assets/ef71c284-994f-4b9e-aeaa-e1e94661f1c1)

### Connecting to Instances

#### Accessing the Bastion Host

```bash
ssh -i Bastion.pem ubuntu@13.232.198.24
```

- Direct access to private instances is not allowed, so the Bastion Host acts as an intermediary.

![Access Bastion](https://github.com/user-attachments/assets/579b3cef-7165-4b76-8a70-e255ecce7071)

#### Connecting to the Controller via Bastion

```bash
ssh -i Controller.pem ubuntu@10.0.1.211
```
![Access Controller from Bastion](https://github.com/user-attachments/assets/7fb12fb6-3f27-498c-8ff2-2fc9a6f45396)

- Secure access to the controller node is only allowed through the Bastion Host to prevent direct exposure.

#### Checking Internet Access on Private Subnet's Controller to Install Slurm

![Checking Internet on Private Node](https://github.com/user-attachments/assets/192686d2-8a3d-432f-ac9d-2199bbead138)

# Automating above manual infrastructure using terraform
## Creating IAM User with Administrator Access

Follow these steps to create an IAM user with administrator access and generate credentials:

## Step 1: Access IAM Console

1. Sign in to AWS Management Console
2. Navigate to IAM (Identity and Access Management) service

## Step 2: Create New User

1. Click on "Users" in the left navigation pane
2. Click "Add users" button
3. Enter a username for the new IAM user
4. Select "Provide user access to the AWS Management Console" if console access is needed
5. Choose password options (auto-generated or custom)

## Step 3: Set Permissions

1. On the permissions page, select "Attach policies directly"
2. Search for and select "AdministratorAccess" policy
3. Review and confirm the selection

## Step 4: Generate Access Keys

1. After user creation, go to the user's security credentials tab
2. Scroll to "Access keys" section
3. Click "Create access key"
4. Select use case (usually "Command Line Interface (CLI)")
5. Acknowledge the recommendations and proceed
6. Download or copy the credentials (Access key ID and Secret access key)

**Important:** Store the access key ID and secret access key securely. The secret access key cannot be retrieved after initial creation.

## Best Practices

- Enable MFA (Multi-Factor Authentication) for additional security
- Regularly rotate access keys
- Follow the principle of least privilege when assigning permissions
- Monitor user activity through AWS CloudTrail

## Project Flow Diagram

```mermaid
graph TD
    A[Start] --> B[Terraform Infrastructure Setup]
    B --> C[Create AWS EC2 Controller]
    B --> D[Create Local Container 1 with GPU]
    B --> E[Create Local Container 2 with GPU]
    
    C --> F[Submit ML Job from Controller]
    F --> G{Job Distribution}
    G --> H[Container 1 Processing]
    G --> I[Container 2 Processing]
    
    H --> J[Results Collection]
    I --> J
    J --> K[Display Output on Controller]
    K --> L[End]
    
    %% Additional details
    C --> M[AWS EC2 Setup]
    M --> N[Security Groups]
    M --> O[Network Config]
    
    D --> P[GPU Configuration]
    E --> P
    P --> Q[CUDA Setup]
    
    F --> R[Job Parameters]
    R --> S[Resource Allocation]
```

## Sequence Diagram

```mermaid
sequenceDiagram
    participant U as User
    participant C as Controller
    participant N as Compute Node
    participant S as Storage
    
    U->>C: Submit Job
    C->>C: Validate Job
    C->>N: Allocate Resources
    N->>S: Access Data
    N->>N: Process Job
    N->>C: Return Results
    C->>U: Display Results

```

## Component Details

- **Controller (AWS EC2):** Manages job submission and result collection
- **Local Containers:** GPU-enabled environments for ML computation
- **Terraform:** Infrastructure as Code for automated setup
- **Job Flow:** Controller → Compute Nodes → Results → Controller

## Terraform Configuration for controller

```jsx
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region     = "ap-south-1"
  access_key = ""
  secret_key = ""
}

resource "aws_vpc" "slurm_vpc" {
  cidr_block = "10.0.0.0/16"
  tags = {
    Name = "SLURM-VPC"
  }
}

resource "aws_subnet" "public_subnet" {
  vpc_id                  = aws_vpc.slurm_vpc.id
  cidr_block              = "10.0.2.0/24"
  map_public_ip_on_launch = true
  availability_zone       = "ap-south-1a"
  tags = {
    Name = "PublicSubnet"
  }
}

resource "aws_subnet" "private_subnet" {
  vpc_id            = aws_vpc.slurm_vpc.id
  cidr_block        = "10.0.1.0/24"
  availability_zone = "ap-south-1a"
  tags = {
    Name = "PrivateSubnet"
  }
}

resource "aws_internet_gateway" "slurm_igw" {
  vpc_id = aws_vpc.slurm_vpc.id
  tags = {
    Name = "SLURM-IGW"
  }
}

resource "aws_route_table" "public_rt" {
  vpc_id = aws_vpc.slurm_vpc.id
  tags = {
    Name = "Public-RT"
  }
}

resource "aws_route" "public_internet_access" {
  route_table_id         = aws_route_table.public_rt.id
  destination_cidr_block = "0.0.0.0/0"
  gateway_id             = aws_internet_gateway.slurm_igw.id
}

resource "aws_route_table_association" "public_assoc" {
  subnet_id      = aws_subnet.public_subnet.id
  route_table_id = aws_route_table.public_rt.id
}

resource "aws_instance" "bastion_host" {
  ami             = "ami-00bb6a80f01f03502"
  instance_type   = "t2.micro"
  subnet_id       = aws_subnet.public_subnet.id
  security_groups = [aws_security_group.controller_sg.name]
  key_name        = aws_key_pair.key_pair.key_name
  tags = {
    Name = "Bastion-Host"
  }
}

resource "aws_instance" "controller" {
  ami             = "ami-00bb6a80f01f03502"
  instance_type   = "t3.medium"
  subnet_id       = aws_subnet.private_subnet.id
  security_groups = [aws_security_group.controller_sg.name]
  key_name        = aws_key_pair.key_pair.key_name
  tags = {
    Name = "SLURM-Controller"
  }
}
```

**Note:** Remember to replace the access_key and secret_key with your own AWS credentials, and ensure they are stored securely. It's recommended to use environment variables or AWS credentials file instead of hardcoding them.

### Key Components:

- **Provider Configuration:** Sets up AWS provider with region and authentication
- **Key Pair Generation:** Creates RSA key pair for EC2 instance access
- **EC2 Instance:** Deploys t2.micro instance with Ubuntu AMI
- **Local File:** Saves private key for SSH access

## Accessing EC2 Instance via CLI

To access your EC2 instance using the PEM file, follow these steps:

1. First, change the permissions of your PEM file to make it secure:
`chmod 400 your-key-name.pem`
2. Use SSH to connect to your EC2 instance:
`ssh -i your-key-name.pem ubuntu@your-instance-public-ip`

For example, using the instance from our configuration:

```bash
chmod 400 controller-key.pem
ssh -i controller-key.pem ubuntu@13.233.71.90
```

**Note:** Replace 'your-key-name.pem' with your actual key file name and 'your-instance-public-ip' with your EC2 instance's public IP address.

### Troubleshooting Tips:

- If connection fails, verify that your security group allows inbound SSH (port 22)
- Ensure you're using the correct username (usually 'ubuntu' for Ubuntu AMIs)
- Check that your instance is running and has a public IP address



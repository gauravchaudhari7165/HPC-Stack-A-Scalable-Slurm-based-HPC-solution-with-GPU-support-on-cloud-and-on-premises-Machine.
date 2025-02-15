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

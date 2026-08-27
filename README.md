# AWS VPC Infrastructure — Production-Style Web Infrastructure

## Overview

This project is the second major project in my **100 Days Cloud AWS/DevOps hands-on learning journey**.

The goal of this project was to build a production-style AWS networking environment using **Terraform Infrastructure as Code (IaC)** and gain practical experience with Amazon VPC networking, subnet design, routing, security groups, EC2, Nginx, and Application Load Balancing.

The infrastructure was designed incrementally to demonstrate how an internet-facing application can be deployed across multiple Availability Zones while separating public and private network resources.

The project also serves as a networking foundation for future projects involving **Docker, Amazon ECS, and AWS Fargate**.

---

## Project Objectives

The primary objectives were to:

* Understand AWS VPC architecture and CIDR planning.
* Create a multi-AZ network across two Availability Zones.
* Understand the difference between public and private subnets.
* Configure Internet Gateway and route tables.
* Understand how routing determines whether a subnet is public or private.
* Configure Security Groups for controlled network access.
* Deploy EC2 instances running Nginx.
* Distribute EC2 instances across multiple Availability Zones.
* Deploy an Application Load Balancer.
* Configure target groups and health checks.
* Demonstrate basic high availability.
* Manage the infrastructure entirely through Terraform.
* Practice troubleshooting AWS networking and infrastructure issues.
* Understand the purpose and cost implications of NAT Gateway.
* Build a foundation for future containerized AWS deployments.

---

## Architecture

The final deployed architecture consists of a VPC spanning two Availability Zones.

```text
                         INTERNET
                            |
                            v
                    Internet Gateway
                            |
                            v
                +-----------------------+
                |   Application Load    |
                |      Balancer         |
                +-----------------------+
                    /               \
                   /                 \
                  v                   v
          Public Subnet AZ-1    Public Subnet AZ-2
           10.0.1.0/24            10.0.2.0/24
                 |                     |
                 v                     v
              EC2 AZ-1              EC2 AZ-2
               Nginx                  Nginx
                 |                     |
                 +----------+----------+
                            |
                       VPC Network
                            |
                +-----------+-----------+
                |                       |
         Private Subnet AZ-1     Private Subnet AZ-2
          10.0.3.0/24             10.0.4.0/24
```

<img width="1920" height="1080" alt="Screenshot (396)" src="https://github.com/user-attachments/assets/35761ae7-3f95-4645-bc55-1062d35e0675" />


<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/4b31d9c0-5a1a-4748-bc7f-8f7e1ff7c978" />


### Network Design

```text
VPC
10.0.0.0/16
│
├── Availability Zone 1
│   ├── Public Subnet
│   │   └── 10.0.1.0/24
│   │
│   └── Private Subnet
│       └── 10.0.3.0/24
│
└── Availability Zone 2
    ├── Public Subnet
    │   └── 10.0.2.0/24
    │
    └── Private Subnet
        └── 10.0.4.0/24
```

---

## AWS Services Used

* **Amazon VPC** — Provides the isolated virtual network.
* **Amazon EC2** — Hosts the Nginx web servers.
* **Application Load Balancer** — Distributes HTTP traffic between EC2 instances.
* **Internet Gateway** — Provides internet connectivity for the VPC's public resources.
* **Route Tables** — Control traffic routing between the VPC, subnets, and internet gateway.
* **Security Groups** — Control inbound and outbound traffic to EC2 instances and the ALB.
* **Amazon Linux 2023** — Operating system for the EC2 instances.
* **Nginx** — Web server running on the EC2 instances.
* **Terraform** — Provisions and manages AWS infrastructure.

---

# Infrastructure Components

## 1. VPC

A VPC was created using the CIDR block:

```text
10.0.0.0/16
```

This provides the overall private IPv4 address space for the infrastructure.

Terraform also enabled DNS support and DNS hostnames:

```hcl
enable_dns_support   = true
enable_dns_hostnames = true
```

These settings are useful for resources such as EC2 instances that rely on VPC DNS functionality.

---

## 2. Availability Zones

The infrastructure uses two Availability Zones.

Terraform retrieves the available Availability Zones dynamically rather than hardcoding specific AZ names.

```hcl
data "aws_availability_zones" "available" {
  state = "available"
}
```

The first two available AZs are used for the project.

Using multiple Availability Zones improves resilience because the application infrastructure is not dependent on a single physical AWS Availability Zone.

---

## 3. Subnet Design

Four subnets were created.

| Subnet       | CIDR          | Purpose                       |
| ------------ | ------------- | ----------------------------- |
| Public AZ-1  | `10.0.1.0/24` | Public-facing resources       |
| Public AZ-2  | `10.0.2.0/24` | Public-facing resources       |
| Private AZ-1 | `10.0.3.0/24` | Private application resources |
| Private AZ-2 | `10.0.4.0/24` | Private application resources |

The subnets are distributed across two Availability Zones.

The public subnets are intended for resources that need direct internet connectivity, such as the Application Load Balancer.

The private subnets are reserved for resources that should not be directly reachable from the internet.

---

# Public vs Private Subnets

One of the major networking concepts demonstrated by this project is that a subnet is not inherently public or private simply because it is named "public" or "private."

Its routing configuration determines its behavior.

### Public subnet

The public subnets use a route table containing:

```text
0.0.0.0/0 → Internet Gateway
```

This provides a route for internet-bound traffic.

### Private subnet

The private subnets use a separate route table containing the VPC local route:

```text
10.0.0.0/16 → local
```

They do not have a direct route to the Internet Gateway.

Therefore, resources in these subnets cannot directly access the internet.

---

# Internet Gateway

An Internet Gateway was attached to the VPC.

Its purpose is to provide a path between the VPC and the internet.

The public route table contains:

```text
Destination: 0.0.0.0/0
Target: Internet Gateway
```

The traffic flow for public resources is therefore:

```text
EC2
 ↓
Public Subnet
 ↓
Public Route Table
 ↓
Internet Gateway
 ↓
Internet
```

<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/5fe07f98-b5c3-4f98-9385-72f6e2433e5d" />

---

# Route Tables

Two route tables were created.

## Public Route Table

The public route table contains:

```text
10.0.0.0/16 → local
0.0.0.0/0   → Internet Gateway
```

It is associated with:

```text
10.0.1.0/24
10.0.2.0/24
```

## Private Route Table

The private route table contains the VPC local route:

```text
10.0.0.0/16 → local
```

It is associated with:

```text
10.0.3.0/24
10.0.4.0/24
```

No direct Internet Gateway route was configured for the private subnets.

---

# Security Groups

Two separate Security Groups were created.

## ALB Security Group

The Application Load Balancer Security Group allows:

| Protocol | Port | Source      |
| -------- | ---: | ----------- |
| TCP      |   80 | `0.0.0.0/0` |

This allows users on the internet to access the load balancer over HTTP.

Outbound traffic is allowed.

---

## EC2 Security Group

The EC2 Security Group allows:

| Protocol | Port | Source                 |
| -------- | ---: | ---------------------- |
| TCP      |   80 | ALB Security Group     |
| TCP      |   22 | Administrator IP `/32` |

This creates an important security boundary.

Instead of allowing HTTP traffic directly from the entire internet to the EC2 instances, the EC2 instances only accept HTTP traffic from the Application Load Balancer.

SSH access is restricted to the administrator's public IP.

This is significantly safer than:

```text
SSH → 0.0.0.0/0
```

<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/b08b84f6-1a64-41a0-81b7-9bd82812c1b9" />

---

# EC2 Web Servers

Two EC2 instances were deployed:

```text
EC2 AZ-1
    |
    └── Nginx

EC2 AZ-2
    |
    └── Nginx
```

Both instances use Amazon Linux 2023.

Nginx is installed automatically using Terraform `user_data`.

The instances were intentionally deployed in different Availability Zones to demonstrate multi-AZ architecture.

Each server also displays its Availability Zone in the web page, making it easy to verify which backend is responding.

<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/04fd4b1c-df5d-44b1-8ec3-28277ad4539c" />

---

# Application Load Balancer

An Application Load Balancer was deployed across both public subnets.

```text
ALB
├── Public AZ-1
└── Public AZ-2
```

The ALB listens on:

```text
HTTP :80
```

and forwards traffic to the web target group.

Traffic flow:

```text
User
  |
  v
Application Load Balancer
  |
  v
Target Group
  |
  +---------> EC2 AZ-1
  |
  +---------> EC2 AZ-2
```

<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/015650ad-18fa-4153-a239-865e0dc0b205" />

<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/43c8b322-592b-4ed1-a64f-7b5e79e16e72" />


---

# Target Group

The target group contains both EC2 instances.

```text
Target Group
├── EC2 AZ-1
└── EC2 AZ-2
```

The target group uses HTTP on port 80.

Health checks are performed against:

```text
/
```

with an expected HTTP response of:

```text
200
```

If an instance becomes unhealthy, the ALB can stop sending traffic to it.

<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/128fcaaa-45cd-4d94-a6bd-b443fc5b6bab" />

<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/ac8dcd24-c16d-42c6-86cb-622c11aca2ef" />



---

# High Availability

The project demonstrates basic high availability by distributing the web servers across two Availability Zones.

```text
                  ALB
                 /   \
                /     \
             AZ-1     AZ-2
              |         |
             EC2       EC2
              |         |
            Nginx     Nginx
```

If one Availability Zone or EC2 instance becomes unavailable, the other instance can continue serving requests through the load balancer, assuming the ALB and remaining target remain healthy.

This is substantially more resilient than running the application on a single EC2 instance.

---

# NAT Gateway Decision

A NAT Gateway was **intentionally not deployed**.

A typical production architecture would allow private EC2 instances to access the internet through:

```text
Private EC2
     |
Private Route Table
     |
NAT Gateway
     |
Internet Gateway
     |
Internet
```

The NAT Gateway would normally be deployed in a public subnet, while private subnets would route internet-bound traffic through it.

However, NAT Gateway introduces additional AWS charges, including hourly and data-processing costs.

Since this project is part of a cost-conscious learning environment, the NAT Gateway was deliberately excluded.

The private route tables therefore do not currently contain:

```text
0.0.0.0/0 → NAT Gateway
```

This was an intentional design decision rather than a missing component.

For a production workload where private instances require outbound internet access, a NAT-based design or suitable AWS-native alternatives should be evaluated.

---

# Networking Connectivity Test

A temporary EC2 instance was deployed into a private subnet to test private subnet behavior.

The instance did not receive a public IP address.

An attempt was made to SSH into the private instance from the public EC2 instance.

The connection timed out.

This behavior demonstrated that private subnet resources were not directly reachable using the current routing and security configuration.

The temporary instance was subsequently removed.

This exercise helped demonstrate the difference between:

* Routing
* Public/private subnet behavior
* Security Group restrictions
* Internet Gateway access
* NAT Gateway requirements

---

# Terraform

The entire infrastructure was provisioned using Terraform.

The project uses a basic Terraform structure:

```text
aws-vpc-infrastructure/
│
├── main.tf
├── variables.tf
├── outputs.tf
├── providers.tf
├── terraform.tfvars
├── .gitignore
└── README.md
```

### File responsibilities

#### `providers.tf`

Defines:

* Terraform version requirements
* AWS provider
* AWS region

#### `main.tf`

Contains AWS infrastructure resources such as:

* VPC
* Subnets
* Internet Gateway
* Route tables
* Route table associations
* Security Groups
* EC2 instances
* Target group
* Application Load Balancer
* ALB listener

#### `variables.tf`

Contains configurable Terraform variables such as:

```text
aws_region
vpc_cidr
admin_cidr
```

#### `outputs.tf`

Exposes useful infrastructure information such as:

```text
VPC ID
VPC CIDR
Subnet IDs
Route table IDs
Security Group IDs
EC2 instance IDs
EC2 public IPs
ALB DNS name
```

#### `terraform.tfvars`

Contains environment-specific variable values.

This file is excluded from Git to prevent accidental exposure of configuration values.

---

# Terraform Workflow

The project follows the standard Terraform workflow:

```text
terraform init
       |
       v
terraform fmt
       |
       v
terraform validate
       |
       v
terraform plan
       |
       v
terraform apply
```

### Initialize

```bash
terraform init
```

Downloads and initializes the required providers.

### Format

```bash
terraform fmt
```

Formats Terraform configuration files.

### Validate

```bash
terraform validate
```

Checks the Terraform configuration for syntax and configuration errors.

### Plan

```bash
terraform plan
```

Shows the infrastructure changes Terraform intends to make before applying them.

### Apply

```bash
terraform apply
```

Creates or modifies the infrastructure.

---

# Troubleshooting and Lessons Learned

Several practical networking and infrastructure concepts were reinforced while building the project.

## 1. Public vs Private Subnets

A subnet is not automatically public because it is named "public."

The route table determines whether it has a route to an Internet Gateway.

### Lesson

```text
Public subnet
→ route to Internet Gateway
```

while:

```text
Private subnet
→ no direct Internet Gateway route
```

---

## 2. Private EC2 Connectivity Timeout

A temporary EC2 instance was deployed in a private subnet.

SSH access to the instance timed out.

This was expected because the instance had no public IP and the private subnet did not have a suitable path for direct internet access.

The test demonstrated why private infrastructure needs controlled access mechanisms such as a bastion host, Systems Manager, VPN, or other appropriate management architecture.

---

## 3. Security Group Separation

Initially, the EC2 Security Group allowed HTTP traffic from anywhere.

Once the ALB was introduced, the configuration was improved so that:

```text
Internet → ALB
```

and:

```text
ALB → EC2
```

The EC2 Security Group therefore only accepts HTTP traffic from the ALB Security Group.

This follows the principle of limiting access to only the required source.

---

## 4. SSH Restriction

SSH access was restricted using the administrator's public IP:

```text
YOUR_IP/32
```

instead of:

```text
0.0.0.0/0
```

This reduces unnecessary exposure of SSH to the internet.

---

## 5. Dynamic Availability Zones

Instead of hardcoding AZ names, Terraform retrieves available AZs dynamically:

```hcl
data "aws_availability_zones" "available" {
  state = "available"
}
```

This makes the Terraform configuration more portable between AWS accounts.

---

## 6. NAT Gateway Cost

The NAT Gateway was deliberately not created because it can generate charges.

The project therefore demonstrates the architecture and purpose of NAT without unnecessarily incurring the cost during the learning phase.

---

# Security Considerations

The project applies several basic security best practices:

* SSH is restricted to a specific administrator IP.
* EC2 HTTP traffic is restricted to the ALB Security Group.
* The ALB and EC2 instances use separate Security Groups.
* Private subnets do not have direct Internet Gateway access.
* Terraform state files are excluded from Git.
* Terraform variable files are excluded from Git.
* SSH private keys are excluded from Git.
* Infrastructure is provisioned through code instead of manual configuration.

---

# Cost Considerations

The project was designed with AWS cost awareness in mind.

Resources such as:

* VPC
* Subnets
* Route tables
* Internet Gateway
* Security Groups

do not carry the same ongoing compute/networking costs as resources such as EC2, ALB, and NAT Gateway.

The major resources requiring attention are:

* EC2 instances
* Application Load Balancer
* NAT Gateway

The NAT Gateway was intentionally not deployed.

When the project is not being actively used, unnecessary billable resources should be terminated or destroyed.

Terraform can be used to clean up the environment:

```bash
terraform destroy
```

Before running `terraform destroy`, always review the Terraform plan carefully because this will remove the infrastructure managed by the configuration.

---

# Final Architecture

The resulting architecture demonstrates:

```text
                              INTERNET
                                  |
                                  v
                         Internet Gateway
                                  |
                                  v
                    +------------------------+
                    | Application Load       |
                    | Balancer               |
                    +------------------------+
                       /                  \
                      /                    \
                     v                      v
              Public Subnet AZ-1     Public Subnet AZ-2
               10.0.1.0/24             10.0.2.0/24
                     |                      |
                     v                      v
                  EC2 AZ-1              EC2 AZ-2
                   Nginx                  Nginx
                     |                      |
                     +----------+-----------+
                                |
                         VPC Internal Network
                                |
                    +-----------+-----------+
                    |                       |
             Private Subnet AZ-1     Private Subnet AZ-2
              10.0.3.0/24             10.0.4.0/24
```

---

# Skills Demonstrated

This project demonstrates practical experience with:

* AWS VPC
* IPv4 CIDR planning
* Subnetting
* Availability Zones
* Public and private subnets
* Internet Gateway
* Route tables
* Route table associations
* Security Groups
* EC2
* Amazon Linux
* Nginx
* Application Load Balancer
* Target Groups
* Health Checks
* Multi-AZ architecture
* Terraform
* Infrastructure as Code
* AWS networking troubleshooting
* Basic AWS security principles
* Cost-aware AWS architecture

---

# Future Improvements

The project can be extended with:

1. NAT Gateway for private subnet outbound connectivity.
2. EC2 instances moved entirely into private subnets.
3. ALB-only access to application servers.
4. AWS Systems Manager Session Manager for instance administration.
5. Network ACL configuration where appropriate.
6. Terraform modules for reusable infrastructure.
7. Remote Terraform state using Amazon S3 and state locking.
8. Better environment separation such as `dev`, `staging`, and `prod`.
9. GitHub Actions CI/CD.
10. GitHub OIDC authentication to AWS.
11. Automated Terraform `plan` and `apply`.
12. Docker containerization.
13. Migration from EC2 to Amazon ECS/Fargate.

The eventual target architecture is to use the networking knowledge from this project as the foundation for deploying containerized workloads on AWS ECS/Fargate.

---

# Project Outcome

This project provided hands-on experience designing and deploying a multi-AZ AWS network rather than simply creating individual AWS resources.

The most important takeaway was understanding how AWS networking components work together:

```text
VPC
 ↓
Subnets
 ↓
Route Tables
 ↓
Internet Gateway
 ↓
Security Groups
 ↓
EC2
 ↓
Application Load Balancer
 ↓
High Availability
```

The project establishes the networking foundation required for more advanced AWS DevOps work, particularly **Docker, ECS, Fargate, CI/CD, and production-style cloud infrastructure**.

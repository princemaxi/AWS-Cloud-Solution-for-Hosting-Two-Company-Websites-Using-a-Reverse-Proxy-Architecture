# AWS Cloud Solution for Hosting Two Company Websites Using a Reverse Proxy Architecture

## 📘 1. Architecture Overview

This project implements a highly available, scalable, secure, and cost-efficient multi-tier architecture on AWS to host two separate company websites behind a central NGINX Reverse Proxy layer.
The architecture is designed to ensure:

- **🔐 Security** (public exposure only through ALB and NGINX)
- **⚡ High availability** (multi-AZ redundancy)
- **📈 Scalability** (Auto Scaling Groups at each tier)
- **🔄 Centralized content sharing** (EFS)
- **🗃️ Reliable transactional database** (RDS Multi-AZ)

The reference architecture diagram used is shown below:

![alt text](/images/a1.png)

---

## 📡 2. High-Level Architecture Components

The solution is deployed inside a dedicated AWS VPC that spans two Availability Zones (AZ-A and AZ-B).
Each AZ contains:

- Public Subnets
- Private Application Subnets
- Private Data Subnets

This enforces security best practices where only the NGINX layer is internet-facing.

---

## 🏗️ 3. Network Architecture (VPC Layer)
### 3.1 Virtual Private Cloud (VPC)

- CIDR: 10.0.0.0/16
- DNS Support: Enabled
- DNS Hostnames: Enabled

The VPC is divided into six subnets across two AZs:

### Public Subnets (AZ-A & AZ-B)

Used for:

- Bastion Hosts
- NGINX Reverse Proxy EC2
- NAT Gateways

### Private Application Subnets (AZ-A & AZ-B)

Used for hosting:

- WordPress Application Servers
- Tooling Application Servers
- Internal Load Balancers

### Private Data Subnets (AZ-A & AZ-B)

Used for:

- EFS
- RDS Multi-AZ Database Layer

---

## 🌍 4. Internet Access & Traffic Flow
### 4.1 Internet Gateway (IGW)

Attached to the VPC, enabling outbound internet for resources in public subnets.

### 4.2 Public NAT Gateways (One per AZ)

Provide outbound internet to private subnets without exposing them publicly.

### 4.3 Route 53 (DNS)

(Optional)
Handles domain-level routing to the public Application Load Balancer (ALB).

---

## 🎛️ 5. Load Balancing & Reverse Proxy Layer
### 5.1 Public Application Load Balancer (ALB)

- Internet-facing
- Terminates HTTP/HTTPS
- Forwards traffic to NGINX target group
- Provides centralized routing
- Supports auto-scaling NGINX pool

### 5.2 NGINX Reverse Proxy Layer

Runs on EC2 Auto Scaling Group in public subnets.

Responsibilities:

- Receives traffic from ALB
- Acts as reverse proxy
- Routes traffic internally to:
  - WordPress ALB
  - Tooling ALB

This isolates the backend websites and prevents direct access from the internet.

---

## 🔁 6. Internal Load Balancers

Each website has its own internal ALB, accessible only from NGINX.

### 6.1 Internal ALB for WordPress

- Scheme: Internal
- Listener: HTTP/80
- Forwards to WordPress ASG target group

### 6.2 Internal ALB for Tooling

- Scheme: Internal
- Listener: HTTP/80
- Forwards to Tooling ASG target group

This separation ensures:

- Isolation between services
- Distributed failure domains
- Independent scaling for each application

--- 

## 🖥️ 7. Compute Layer (EC2 Auto Scaling)
### 7.1 Bastion Hosts

- Located in Public Subnets
- SSH access restricted to admin IP
- Used to securely access private EC2 instances

###  7.2 NGINX Reverse Proxy ASG

- Located in public subnets
- Scales horizontally
- Communicates with internal ALBs only

### 7.3 WordPress ASG

- Located in private subnets
- Integrated with EFS
- Uses internal ALB-WordPress

### 7.4 Tooling ASG

- Located in private subnets
- Also mounted to EFS
- Uses internal ALB-Tooling

--- 

## 📂 8. Shared Storage Layer (EFS)

Amazon Elastic File System is used to:

- Store shared content
- Allow both NGINX and Webservers to access uploads, logs, configs

EFS has:

- Two mount targets (one per AZ)
- Attached to NGINX and Webserver SG via port 2049 (NFS)

---

## 🗄️ 9. Database Layer (Amazon RDS)
### RDS MySQL/PostgreSQL

- Multi-AZ deployment
- Subnet group includes private data subnets
- Encrypted with a customer-managed KMS key
- SG allows only app servers to connect on port 3306

This ensures:

- High availability
- Automated failover
- Backup retention

---

### 🔐 10. Security Architecture

The entire system uses Security Groups with least-privilege access:

| Flow                          | Description                     | Ports  |
| ----------------------------- | ------------------------------- | ------ |
| **Internet → ALB Public**     | Public entry point              | 80/443 |
| **ALB Public → NGINX**        | Reverse proxy tier              | 80/443 |
| **NGINX → Internal ALB**      | Routing websites internally     | 80     |
| **Internal ALB → Webservers** | Delivery of application content | 80     |
| **Webservers → RDS**          | Database traffic                | 3306   |
| **NGINX/Webservers → EFS**    | Shared file system              | 2049   |
| **Bastion → Private EC2**     | Admin SSH                       | 22     |


No other component is exposed publicly — strict Zero-Trust boundaries.

---

## 📦 11. Summary of Architecture Benefits
| Feature             | Benefit                                  |
| ------------------- | ---------------------------------------- |
| Multi-AZ            | High Availability                        |
| Auto Scaling        | Elastic Performance                      |
| NGINX Reverse Proxy | Central routing, isolates backend        |
| Internal ALBs       | Service separation & security            |
| EFS                 | Shared storage between tiers             |
| RDS Multi-AZ        | Reliable transactional database          |
| Bastion             | Secure access to private servers         |
| NAT Gateways        | Safe outbound access for private subnets |

---

# IMPLEMENTATION
# 📌 Phase 1 — Prerequisite 

This phase outlines all the required setup steps before building the full infrastructure.

## 1. Configure AWS Accounts and Organizational Structure

To follow best practices and isolate workloads, you will use:

- One AWS Master (Root) Account
- One Child Account called `DevOps`
- One Organizational Unit (OU) called `Dev`

### Step 1: Create AWS Master Account

If you do not already have an AWS account:

1. Go to: https://aws.amazon.com
2. Sign up using your primary email address
3. Add required billing details
4. Complete identity verification

This becomes your **Root Account**. You will NOT deploy infrastructure here—only manage organizations.

### Step 2: Create an AWS Organizational Unit (OU)

Inside the Master account:

1. Open AWS Organizations
2. Select Create Organizational Unit
3. Name the OU: `Dev`
4. This OU will contain development-related AWS accounts.
![alt text](/images/1.png)
![alt text](/images/2.png)
![alt text](/images/3.png)
![alt text](/images/4.png)
![alt text](/images/5.png)
![alt text](/images/6.png)
![alt text](/images/7.png)

### Step 3: Create a New AWS Sub-Account (`DevOps`)

Still in the Master account:

1. Go to **AWS Organizations → Accounts → Add Account**
2. Choose **Create AWS Account**
3. Enter:
   - Account name: DevOps
   - Email address: A different email from the Master account (You must have access to this email)

4. AWS will create the account automatically.
![alt text](/images/8.png)
![alt text](/images/9.png)

### Step 4: Move the DevOps Account into the Dev OU

1. In AWS Organizations
2. Select the **DevOps** account
3. Click **Move**
4. Choose the OU: Dev
![alt text](/images/10.png)
![alt text](/images/11.png)
![alt text](/images/12.png)

This places all your development resources under proper governance.

### Step 5: Log in to the DevOps Account

Now sign in as the administrator of the DevOps account:

1. Open the verification email you received

2. Log in with:
   - The new email
   - New password you create
3.  This **DevOps** account is where you will deploy all infrastructure in the project.
![alt text](/images/13.png)

## 2. Domain Name (Optional — Skipped in This Project)

Normally, the project requires:

- ✔ Getting a domain
- ✔ Creating a Route53 Hosted Zone
- ✔ Linking your domain to Route53 nameservers

**BUT since we are NOT using a domain, this entire section is skipped.**

Our environment will function using:

- **ALB DNS Name**
- **Private DNS within VPC**
- **Nginx reverse proxy using paths instead of domains (optional)**

Example:
```arduino
http://ALB-PUBLIC-DNS/site1
http://ALB-PUBLIC-DNS/site2
```
No DNS configuration is required.

---

# 🧩 Phase 2 — Networking & VPC Implementation

This phase walks through the complete creation of the Virtual Private Cloud (VPC) and all networking components required to support the multi-website architecture.
All configurations strictly align with the architectural diagram and the CIDR plan below.

## 📌 CIDR Allocation (Aligned With Architecture)
| Layer                                    | AZ-A CIDR       | AZ-B CIDR   |
| ---------------------------------------- | --------------- | ----------- |
| **Public Subnets** (Bastion, NGINX, NAT) | 10.0.1.0/24     | 10.0.3.0/24 |
| **Private App Subnets** (Webservers)     | 10.0.2.0/24     | 10.0.4.0/24 |
| **Private Data/EFS Subnets** (DB/EFS)    | 10.0.5.0/24     | 10.0.6.0/24 |
| **VPC CIDR**                             | **10.0.0.0/16** |             |	

This layout provides multi-AZ high availability and separation of public, application, and data layers.

🔷 Step-by-Step VPC Implementation
### 1. Create a VPC

**Console:** AWS Console → VPC → Your VPCs → Create VPC

- Name: multiweb-vpc
- CIDR: 10.0.0.0/16
- Tenancy: default

![alt text](/images/V1.png)
![alt text](/images/V2.png)
![alt text](/images/V3A.png)
![alt text](/images/V3B.png)

### 2. Create the Subnets (6 Total)

Create all subnets according to the CIDR plan:

**AZ-A:**

- Public Subnet → 10.0.1.0/24
- Private App Subnet → 10.0.2.0/24
- Private Data/EFS Subnet → 10.0.5.0/24


**AZ-B:**

- Public Subnet → 10.0.3.0/24
- Private App Subnet → 10.0.4.0/24
- Private Data/EFS Subnet → 10.0.6.0/24

![alt text](/images/V4.png)
![alt text](/images/V7.png)
![alt text](/images/V8.png)
![alt text](/images/V9.png)
![alt text](/images/V10.png)

**Important:** Enable Auto-Assign Public IPv4 on both public subnets.
![alt text](/images/V11.png)

### 3. Create a Route Table for Public Subnets

Name: `rtb-public`

Associate:

- 10.0.1.0/24 (public-azA)
- 10.0.3.0/24 (public-azB)

Add routing to IGW later (after IGW creation).

![alt text](/images/V12.png)
![alt text](/images/V13.png)
![alt text](/images/V14.png)
![alt text](/images/V15.png)
![alt text](/images/V16.png)

### 4. Create a Route Table for Private Subnets

Name: `rtb-private`

Associate:

- Private App Subnets (10.0.2.0/24 & 10.0.4.0/24)
- Private Data Subnets (10.0.5.0/24 & 10.0.6.0/24)

No IGW route here — these will route to NAT Gateway.

![alt text](/images/V17.png)
![alt text](/images/V8.png)
![alt text](/images/V19.png)

### 5. Create an Internet Gateway

Name: `igw-multiweb`

Attach it to the VPC (`multiweb-vpc`).

### 6. Update Public Route Table (Enable Internet Access)

In rtb-public, add:

- Destination: `0.0.0.0/0`
- Target: `igw-multiweb`

This makes public subnets internet-accessible.

![alt text](/images/V20.png)
![alt text](/images/V21.png)
![alt text](/images/V22.png)
![alt text](/images/V23.png)
![alt text](/images/V24.png)

### 7. Allocate 3 Elastic IPs

You need 3 Elastic IPs:

1. For NAT Gateway
2. For Bastion Host (AZ-A)
3. For Bastion Host (AZ-B)

Tag them:

- EIP-NAT
- EIP-Bastion-A
- EIP-Bastion-B

![alt text](/images/V27.png)
![alt text](/images/V28.png)
![alt text](/images/V29.png)

### 8. Create a NAT Gateway

Create NAT Gateway in public subnet AZ-A:

- **Subnet:** `10.0.1.0/24`
- **Elastic IP:** `EIP-NAT`

Update private route table `rtb-private`:

- **Destination:** `0.0.0.0/0`
- **Target:** NAT Gateway ID

![alt text](/images/V30.png)
![alt text](/images/V31.png)
![alt text](/images/V32.png)

This gives private subnets outbound internet access (updates, package installs, etc.).

### 9. Create All Required Security Groups

Below are the SGs required for this phase.
Final tightening of rules will occur after resources (ALB, NGINX, ASGs) are created.

**🟧 A. sg-nginx (NGINX Reverse Proxy Layer — in Public Subnets)**

Inbound (temporary placeholders until ALB exists):

- HTTP 80 → `0.0.0.0/0` (later restrict to sg-alb)
- HTTPS 443 → `0.0.0.0/0` (later restrict to sg-alb)

Outbound:

- Allow all (or later restrict to webserver ports)

**🟩 B. sg-bastion (SSH Entry Point)**

Inbound:

- SSH 22 → Your workstation public IP
  - Find IP: `curl https://ifconfig.me`

Outbound:

- Allow all

**🟦 C. sg-alb (External Application Load Balancer)**

Inbound:

- HTTP 80 → `0.0.0.0/0`
- HTTPS 443 → `0.0.0.0/0` (if you later add ACM certificates)

Outbound:

- To sg-nginx only (enforced later via target group)

**🟧 D. sg-web (Internal Web/App Servers)**

Inbound:

- App port (e.g., 80) → from `sg-nginx`
- SSH 22 → from `sg-bastion`

Outbound:

- MySQL 3306 → `sg-rds`
- NFS 2049 → `sg-efs`

**🟦 E. sg-rds (Database Layer)**

Inbound:

- MySQL 3306 → `sg-web` only

Outbound:

- Allow all (default)

**🟨 F. sg-efs (EFS Mount Targets)**

Inbound:

- NFS 2049 → `sg-nginx`
- NFS 2049 → `sg-web`

Outbound:

- Allow all

![alt text](/images/V35.png)
![alt text](/images/V36A.png)
![alt text](/images/V36B.png)

---

# Phase 3 — Compute Resources Provisioning

This section documents the full Compute Layer setup for the project environment. It includes Nginx reverse proxy servers, Bastion hosts, and Webservers (WordPress + Tooling). All components are deployed across two Availability Zones for high availability.


## 1. Overview of Compute Resources

The compute layer consists of the following components:

* **Nginx Reverse Proxy Instances** in public subnets
* **Bastion Hosts** for secure SSH access
* **Webservers (WordPress + Tooling)** in private subnets
* **Launch Templates** for automated and consistent EC2 provisioning
* **Target Groups** to route traffic via ALBs
* **Auto Scaling Groups (ASG)** for resilience and automated capacity management

## 2. Nginx Reverse Proxy Layer

The Nginx layer handles all inbound application traffic and forwards it to internal load balancers.

## A. Provision EC2 Instances for Nginx

Create two Nginx EC2 instances, one in each public subnet.

### Steps

1. Go to **EC2 → Launch Instance**.
2. Select **Amazon Linux**.
3. Instance type: **t3.micro**.
4. Network configuration:

   * VPC: *Your Project VPC(multiweb-vpc)*
   * Subnet: Public Subnet-1 for the first instance, Public Subnet-2 for the second.
5. Assign security group: **SG-Nginx**.
    ![alt text](/images/C1.png)
    ![alt text](/images/C2.png)
    ![alt text](/images/C3.png)
    ![alt text](/images/C4.png)
6. Install essential packages:

   ```bash
   sudo yum install python3 net-tools vim wget telnet htop -y
   ```
    ![alt text](/images/C5.png)
    ![alt text](/images/C6.png)
    ![alt text](/images/C6B.png)

7. Create an AMI from the configured instance named: **Nginx-AMI-v1**.
    ![alt text](/images/C6C.png)
    ![alt text](/images/C6D.png)

---

## B. Prepare Launch Templates for Nginx (One per AZ)

You will create two launch templates based on the Nginx AMI.

Launch Templates:

* **LT-Nginx-AZ-a**
* **LT-Nginx-AZ-b**

### Required Settings

* AMI: **Nginx-AMI-v1**
* Instance type: **t2.micro**
* Security Group: **SG-Nginx**
* Subnet: *Not specified* (ASG will choose)
![alt text](/images/C7.png)
![alt text](/images/C8A.png)
![alt text](/images/C8B.png)

### User Data (Installs and starts Nginx)

```bash
#!/bin/bash
yum update -y
yum install nginx -y
systemctl enable nginx
systemctl start nginx
```

![alt text](/images/C8C.png)
---

## C. Configure Target Group for Nginx

This Target Group is used by the ALB to forward HTTPS traffic to Nginx.

### Settings

* Target type: **Instances**
* Protocol: **HTTPS**
* Port: **443/80**
* Health check path: **/healthstatus**
* Name: **TG-Nginx-443**
![alt text](/images/C9.png)
![alt text](/images/C9B.png)
![alt text](/images/C9C.png)

Register the two Nginx EC2 instances.
![alt text](/images/C9D.png)
![alt text](/images/C9E.png)
---

## D. Create Auto Scaling Group for Nginx

ASG ensures constant availability of Nginx servers.

### Settings

* Launch Template: **LT-Nginx-AZ-a** (multi-AZ enabled)
* Subnets: Public Subnet-1 & Public Subnet-2
* Target Group: **TG-Nginx-443**
* Desired capacity: **2**
* Minimum capacity: **2**
* Maximum capacity: **4**
* Health checks: **EC2 + ALB**
* Scaling policy: Scale out when **CPU ≥ 90%**
* SNS topic for notifications: **SNS-Nginx-Scaling**

    ![alt text](/images/C10.png)
    ![alt text](/images/C10B.png)
    ![alt text](/images/C10C.png)
    ![alt text](/images/C10D.png)
    ![alt text](/images/C10E.png)
    ![alt text](/images/C10F.png)
    ![alt text](/images/C10Gpng.png)
    ![alt text](/images/C11.png)

**Auto-scalling at work***
![alt text](/images/C11Bpng.png)
---

# 3. Bastion Layer (SSH Entry Point)

Bastion hosts provide secure administrative access to Nginx and Webservers.

## A. Provision EC2 Instances for Bastion

Create two Bastion servers, each in a public subnet.

### Steps

1. Launch Amazon Linux EC2 instance in Public Subnet-1.
2. Assign security group **SG-Bastion** (allows SSH only from admin IP).
![alt text](/images/C12A.png)
![alt text](/images/C12B.png)
![alt text](/images/C13A.png)
![alt text](/images/C13B.png)
3. Install software:

   ```bash
   sudo yum install python3 net-tools vim wget telnet htop -y
   ```
    ![alt text](/images/C14.png)

4. Associate an Elastic IP with each instance.
![alt text](/images/C15.png)
![alt text](/images/C15B.png)
5. Create AMI: **Bastion-AMI-v1**.

---

## B. Launch Templates for Bastion

Create one launch template per AZ.

Launch Templates:

* **LT-Bastion-AZ-a**
* **LT-Bastion-AZ-b**

User Data:

```bash
#!/bin/bash
yum update -y
yum install ansible git -y
```
![alt text](/images/C16A.png)
![alt text](/images/C16B.png)
![alt text](/images/C16C.png)
---

## C. Configure Target Group for Bastion

Used only for health checks.

### Settings

* Target type: **Instances**
* Protocol: **TCP**
* Port: **22**
* Name: **TG-Bastion-22**
![alt text](/images/C17A.png)
![alt text](/images/C17B.png)
![alt text](/images/C17C.png)
---

## D. Auto Scaling Group for Bastion

* Launch Template: LT-Bastion-AZ-a
* Subnets: Public Subnet-1 & 2
* Target Group: TG-Bastion-22
* Desired / Min / Max: **2 / 2 / 4**
* Health checks: EC2 + ALB
* Scaling policy: CPU ≥ 90%
* SNS Topic: **SNS-Bastion-Scaling**
![alt text](/images/C18A.png)
![alt text](/images/C19.png)
---

# 4. Webserver Layer

This layer contains:

* **WordPress EC2 instances**
* **Tooling website EC2 instances**

Deployed across two private subnets.

## A. Provision EC2 Instances for Webservers

You will create:

* 2 × WordPress servers (AZ-a & AZ-b)
* 2 × Tooling servers (AZ-a & AZ-b)

### Steps

1. Launch CentOS instance into Private Subnet-1.
2. Install required packages:

   ```bash
   sudo yum install python3 chrony net-tools vim wget telnet htop php php-cli php-common -y
   ```
![alt text](/images/C20.png)
![alt text](/images/C21png.png)
![alt text](/images/C22.png)
![alt text](/images/C23.png)

3. Create AMIs:

   * **Webserver-WordPress-AMI-v1**
   * **Webserver-Tooling-AMI-v1**

---

## B. Prepare Launch Templates

You will create four templates:

| Website   | AZ | Template Name   |
| --------- | -- | --------------- |
| WordPress | a  | LT-WP-AZ-a      |
| WordPress | b  | LT-WP-AZ-b      |
| Tooling   | a  | LT-Tooling-AZ-a |
| Tooling   | b  | LT-Tooling-AZ-b |

### User Data — WordPress

```bash
#!/bin/bash
yum update -y
yum install httpd php php-mysqlnd -y
systemctl enable httpd
systemctl start httpd
wget https://wordpress.org/latest.tar.gz
tar -xzf latest.tar.gz -C /var/www/html --strip-components=1
chown -R apache:apache /var/www/html
```

### User Data — Tooling

```bash
#!/bin/bash
yum update -y
yum install httpd php -y
systemctl enable httpd
systemctl start httpd
echo "Tooling Website" > /var/www/html/index.html
```
![alt text](/images/C24.png)

---

# 5. Security Considerations

* Only Bastion hosts are accessible from the internet on SSH.
* ALB is the only public entry point for web traffic.
* All internal traffic flows through VPC security groups and private subnets.
* EC2 instances should not have public IPs except Bastion.

---

# Phase 4: TLS, Load Balancers, EFS, RDS

## TLS Configuration (No Domain)

Because no public domain name is available, ACM cannot issue a public certificate. To keep the architecture functional:

- The Public ALB uses the default AWS certificate.
- Internal communication remains HTTP since it is inside the private VPC.

---

## Create the Public ALB for Nginx

This ALB exposes the Nginx reverse proxy to the internet.

### Steps:

* EC2 → Load Balancers → Create Load Balancer
* Select Application Load Balancer
![alt text](/images/Z1.png)
**Configure:**

* Name: ALB-Nginx-Public
* Scheme: Internet-facing
* Subnets: Public Subnet-1 & Public Subnet-2
* Security Group: SG-ALB-Public (Allow 80/443 from anywhere)

**Listeners:**

* HTTP : 80 → Redirect to HTTPS
* HTTPS : 443 → Forward to TG-Nginx-443 (use default AWS certificate). Use when you have a domain name.

**Target Group:**

* Select TG-Nginx-443
![alt text](/images/Z2.png)

**Test Application Load Balancer**
![alt text](/images/Z3.png)

---

## Internal ALBs for WordPress and Tooling

Nginx routes traffic to these ALBs using private networking.

### A. WordPress Internal ALB

* Create ALB: ALB-WordPress-Internal
* Scheme: Internal
* Subnets: Private Subnet-1 & 2
* Security Group: SG-ALB-Internal (allow only from SG-Nginx)
* Listener: HTTP 80 → TG-WordPress-80
* Health check path: /
* Success codes: 200-399
![alt text](/images/Z4.png)
### B. Tooling Internal ALB

Same steps as WordPress but forward to TG-Tooling-80.
![alt text](/images/Z5.png)

---

## Setup EFS

Provides shared storage for Nginx and webservers.

### Steps:

* Create filesystem: EFS-Shared-Content
* Create mount targets in Private Data Subnet-1 & 2
* Attach SG-DataLayer (Allow port 2049 from SG-Nginx & SG-Webservers)
* Create Access Point: EFS-AccessPoint-Shared
![alt text](/images/Z6A.png)
![alt text](/images/Z6B.png)
![alt text](/images/Z6C.png)
![alt text](/images/Z6D.png)
![alt text](/images/Z6E.png)
![alt text](/images/Z6Fpng.png)
![alt text](/images/Z6G.png)
![alt text](/images/Z6H.png)
---

## Setup RDS (MySQL)

### A. Create KMS Key

* KMS → Create Key → Symmetric
* Name: KMS-RDS-Key
![alt text](/images/Z7.png)
![alt text](/images/Z7B.png)
![alt text](/images/Z7C.png)

### B. Create RDS Subnet Group

* RDS → Subnet Groups → Create
* Name: RDS-SubnetGroup
![alt text](/images/Z8.png)
![alt text](/images/Z8B.png)

**Add:**

* Private Data Subnet-1
* Private Data Subnet-2
![alt text](/images/Z8C.png)

### C. Launch RDS Instance

* RDS → Create Database → MySQL 8.x
* DB Instance: db.t3.micro
* Subnet group: RDS-SubnetGroup
* Public access: No
* Security: Allow SG-Webservers → 3306
* Storage encryption: KMS-RDS-Key
![alt text](/images/Z9.png)
![alt text](/images/Z9B.png)
![alt text](/images/Z9C.png)

**Credentials:**

* Username: admin
* Password: Database1123
![alt text](/images/Z9D.png)
![alt text](/images/Z9E.png)
![alt text](/images/Z11.png)

---

# Phase 5: Testing the Architecture

Once all components (ALBs, ASGs, Nginx, WordPress, Tooling, EFS, and RDS) have been deployed, perform the following tests to validate the full architecture.

## Retrieve ALB Public DNS Name

**Go to:**

EC2 → Load Balancers → ALB-Nginx-Public → Description

Copy the DNS name, which looks like:

```
http://ALB-Nginx-Public-123456.region.elb.amazonaws.com
```

This will be used to access both WordPress and Tooling.

## Test WordPress Website

**Open the ALB URL in a browser:**

```
http://ALB-Nginx-Public-123456.region.elb.amazonaws.com
```

**Expected Result:**

* WordPress homepage loads
* No direct access to private EC2 instances
* Nginx successfully routes traffic to ALB-WordPress-Internal → WordPress ASG
![alt text](/images/ZZ1.png)
## Test Tooling Application

**Access via the Nginx path-based routing:**

```
http://ALB-Nginx-Public-123456.region.elb.amazonaws.com/tooling
```

**Expected Result:**

* Tooling application interface loads
* Request flows through Nginx → ALB-Tooling-Internal → Tooling ASG
![alt text](/images/ZZ2.png)
---

# Tips / Troubleshooting

* Confirm both internal target groups are healthy:
  * TG-WordPress-80 → all “healthy”
  * TG-Tooling-80 → all “healthy”
* Validate Storage and Database
  * From any Nginx or Webserver instance:
    * EFS Test: `df -h | grep efs`
    * Should show EFS-Shared-Content mounted.
    * RDS Test (from Webserver): `mysql -h <RDS-endpoint> -u admin -p`
    * Should authenticate successfully.
* Validate High Availability
  * Terminate one instance in each Auto Scaling Group:
    * Nginx ASG
    * WordPress ASG
    * Tooling ASG
    * Expected:
      * New instances are launched automatically
      * ALB health checks turn healthy after bootstrapping
      * Websites remain available
# 3-Tier Infrastructure Deployment on AWS Using Terraform Modules and Ansible Automation

## Project Overview

This project demonstrates the deployment of a complete 3-Tier Web Application Architecture on AWS using Terraform Modules and Ansible Automation.

The architecture consists of:

- Web Tier (Public Subnet)
- Application Tier (Private Subnet)
- Database Tier (Private Subnet)

The project follows Infrastructure as Code (IaC) principles using Terraform and configuration management using Ansible.

The deployed application allows users to submit registration details through a web form hosted on Nginx. The data is processed by a PHP application running on the Application Server and stored in an Amazon RDS MySQL database.

## Final Architecture


![](/img/Screenshot%202026-06-05%20191348.png)
![](/img/Screenshot%202026-06-05%20190954.png)

```
Internet
    │
    ▼
Internet Gateway
    │
    ▼
Public Route Table
    │
    ▼
┌─────────────────────────────────────┐
│ Public Subnet AZ1                   │
│                                     │
│ Web Server (Nginx)                  │
│ Registration Form                   │
│ Bastion Host                        │
└──────────────┬──────────────────────┘
               │
               │ HTTP (Reverse Proxy)
               ▼
┌─────────────────────────────────────┐
│ Private App Subnet AZ1              │
│                                     │
│ Apache + PHP                        │
│ submit.php                          │
└──────────────┬──────────────────────┘
               │
               │ MySQL (Port 3306)
               ▼
┌─────────────────────────────────────┐
│ Private DB Subnet AZ2               │
│                                     │
│ Amazon RDS MySQL                    │
└─────────────────────────────────────┘

Private Subnets
      │
      ▼
NAT Gateway
      │
      ▼
Internet
```

---

## Phase 1 — AWS Account Preparation

### Step 1: Create IAM User for Terraform

> ⚠️ **Never use the root account for Terraform deployments.**

**Navigate to:** AWS Console → IAM → Users → Create User

| Setting | Value |
|---|---|
| Username | `terraform-user` |
| Console Access | Enabled |
| Programmatic Access | Enabled |

**Attach Policy:** `AdministratorAccess`

After creating the user, download and securely save:
- Access Key
- Secret Key

![](/img/Screenshot%202026-06-05%20080915.png)
---

## Phase 2 — Launch Terraform Management Server

This server is where all Terraform and Ansible commands will run.

**EC2 Configuration:**

| Setting | Value |
|---|---|
| Name | `Terraform-Server` |
| AMI | Ubuntu Server 24.04 |
| Instance Type | t2.micro |
| Key Pair | `terraform-key` |
| Storage | 20 GB gp3 |

**Security Group — Inbound Rules:**

| Type | Port | Source |
|---|---|---|
| SSH | 22 | My IP |

**Outbound:** All Traffic

![](/img/Screenshot%202026-06-05%20075958.png)

After launching, connect via SSH:

```bash
ssh -i terraform-key.pem ubuntu@<PUBLIC-IP>
```
![](/img/Screenshot%202026-06-05%20080317.png)

---

## Phase 3 — Install Required Software

### Update the Server

```bash
sudo apt update -y
sudo apt upgrade -y
```

### Install AWS CLI

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o awscliv2.zip
sudo apt install unzip -y
unzip awscliv2.zip
sudo ./aws/install
```
![](/img/Screenshot%202026-06-05%20080414.png)

Verify:

```bash
aws --version
```
![](/img/Screenshot%202026-06-05%20080517.png)

### Configure AWS CLI

```bash
aws configure
```

Enter:
- Access Key
- Secret Key
- Region: `ap-south-1`
- Output: `json`

Verify your identity:

```bash
aws sts get-caller-identity
```
![](/img/Screenshot%202026-06-05%20081132.png)

### Install Terraform

```bash
wget https://releases.hashicorp.com/terraform/1.11.4/terraform_1.11.4_linux_amd64.zip
unzip terraform_1.11.4_linux_amd64.zip
sudo mv terraform /usr/local/bin/
```
![](/img/Screenshot%202026-06-05%20081237.png)

Verify:

```bash
terraform version
```
![](/img/Screenshot%202026-06-05%20081259.png)

### Install Ansible

```bash
sudo apt install ansible -y
```
![](/img/Screenshot%202026-06-05%20081459.png)

Verify:

```bash
ansible --version
```
![](/img/Screenshot%202026-06-05%20081527.png)

---

## Phase 4 — Network Design

### Architecture Overview

```
Internet
   │
   ▼
Web Server (Public Subnet)
   │
   ▼
App Server (Private Subnet)
   │
   ▼
RDS MySQL (Private Subnet)
```

### VPC CIDR

```
10.0.0.0/16  →  65,536 IP addresses (10.0.x.x)
```

### Subnet Plan

| Subnet | Name | CIDR | AZ | Purpose |
|---|---|---|---|---|
| Public 1 | Public-AZ1 | 10.0.1.0/24 | ap-south-1a | Web Server, NAT Gateway |
| Public 2 | Public-AZ2 | 10.0.2.0/24 | ap-south-1b | Future Expansion / HA |
| Private App 1 | App-AZ1 | 10.0.3.0/24 | ap-south-1a | Application Server |
| Private App 2 | App-AZ2 | 10.0.4.0/24 | ap-south-1b | HA / Requirement |
| Private DB 1 | DB-AZ1 | 10.0.5.0/24 | ap-south-1a | RDS Subnet Group |
| Private DB 2 | DB-AZ2 | 10.0.6.0/24 | ap-south-1b | RDS Subnet Group |

---

## Phase 5 — Create Terraform Project Structure

SSH into the Terraform Server and create the project:

```bash
mkdir terraform-3tier
cd terraform-3tier

mkdir modules
mkdir modules/vpc
mkdir modules/security-group
mkdir modules/ec2
mkdir modules/rds

mkdir ansible
mkdir web-files
mkdir app-files

touch provider.tf main.tf variables.tf outputs.tf terraform.tfvars
```

Expected directory tree:

```
terraform-3tier/
├── provider.tf
├── main.tf
├── variables.tf
├── outputs.tf
├── terraform.tfvars
├── modules/
│   ├── vpc/
│   ├── security-group/
│   ├── ec2/
│   └── rds/
├── web-files/
├── app-files/
└── ansible/
```

![](/img/Screenshot%202026-06-05%20082912.png)

---

## Phase 6 — Create AWS Key Pair

**Navigate to:** AWS Console → EC2 → Key Pairs → Create Key Pair

| Setting | Value |
|---|---|
| Name | `3tier-key` |
| Type | RSA |
| Format | PEM |

Download `3tier-key.pem` and transfer it to the Terraform Server:

```bash
scp -i terraform-key.pem \
  3tier-key.pem \
  ubuntu@<TerraformServerIP>:/home/ubuntu/
```
![](/img/Screenshot%202026-06-05%20083649.png)

---

## Phase 7 — VPC Module

### `provider.tf` (Root)

```hcl
terraform {
  required_version = ">= 1.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "us-east-1"
}
```
![](/img/Screenshot%202026-06-05%20085331.png)

### `modules/vpc/variables.tf`

```hcl
variable "vpc_cidr" {
  default = "10.0.0.0/16"
}
```
![](/img/Screenshot%202026-06-05%20085533.png)

### `modules/vpc/main.tf`

```hcl
# VPC
resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = {
    Name = "3tier-vpc"
  }
}

# INTERNET GATEWAY
resource "aws_internet_gateway" "igw" {
  vpc_id = aws_vpc.main.id

  tags = {
    Name = "3tier-igw"
  }
}

# PUBLIC SUBNET 1
resource "aws_subnet" "public_az1" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = "10.0.1.0/24"
  availability_zone       = "us-east-1a"
  map_public_ip_on_launch = true

  tags = {
    Name = "Public-AZ1"
  }
}

# PUBLIC SUBNET 2
resource "aws_subnet" "public_az2" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = "10.0.2.0/24"
  availability_zone       = "us-east-1b"
  map_public_ip_on_launch = true

  tags = {
    Name = "Public-AZ2"
  }
}

# APP SUBNET AZ1
resource "aws_subnet" "app_az1" {
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.3.0/24"
  availability_zone = "us-east-1a"

  tags = {
    Name = "App-AZ1"
  }
}

# APP SUBNET AZ2
resource "aws_subnet" "app_az2" {
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.4.0/24"
  availability_zone = "us-east-1b"

  tags = {
    Name = "App-AZ2"
  }
}

# DB SUBNET AZ1
resource "aws_subnet" "db_az1" {
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.5.0/24"
  availability_zone = "us-east-1a"

  tags = {
    Name = "DB-AZ1"
  }
}

# DB SUBNET AZ2
resource "aws_subnet" "db_az2" {
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.6.0/24"
  availability_zone = "us-east-1b"

  tags = {
    Name = "DB-AZ2"
  }
}

# ELASTIC IP
resource "aws_eip" "nat_eip" {
  domain = "vpc"
}

# NAT GATEWAY
resource "aws_nat_gateway" "nat" {
  allocation_id = aws_eip.nat_eip.id
  subnet_id     = aws_subnet.public_az1.id

  depends_on = [aws_internet_gateway.igw]

  tags = {
    Name = "3tier-nat"
  }
}

# PUBLIC ROUTE TABLE
resource "aws_route_table" "public_rt" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.igw.id
  }

  tags = {
    Name = "Public-RT"
  }
}

# PRIVATE ROUTE TABLE
resource "aws_route_table" "private_rt" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.nat.id
  }

  tags = {
    Name = "Private-RT"
  }
}

# ROUTE TABLE ASSOCIATIONS
resource "aws_route_table_association" "public1" {
  subnet_id      = aws_subnet.public_az1.id
  route_table_id = aws_route_table.public_rt.id
}

resource "aws_route_table_association" "public2" {
  subnet_id      = aws_subnet.public_az2.id
  route_table_id = aws_route_table.public_rt.id
}

resource "aws_route_table_association" "app1" {
  subnet_id      = aws_subnet.app_az1.id
  route_table_id = aws_route_table.private_rt.id
}

resource "aws_route_table_association" "app2" {
  subnet_id      = aws_subnet.app_az2.id
  route_table_id = aws_route_table.private_rt.id
}

resource "aws_route_table_association" "db1" {
  subnet_id      = aws_subnet.db_az1.id
  route_table_id = aws_route_table.private_rt.id
}

resource "aws_route_table_association" "db2" {
  subnet_id      = aws_subnet.db_az2.id
  route_table_id = aws_route_table.private_rt.id
}
```
![](/img/Screenshot%202026-06-05%20085633.png)

### `modules/vpc/outputs.tf`

```hcl
output "vpc_id" {
  value = aws_vpc.main.id
}

output "public_subnet_1" {
  value = aws_subnet.public_az1.id
}

output "app_subnet_1" {
  value = aws_subnet.app_az1.id
}

output "db_subnet_1" {
  value = aws_subnet.db_az1.id
}

output "db_subnet_2" {
  value = aws_subnet.db_az2.id
}
```
![](/img/Screenshot%202026-06-05%20085710.png)

---

## Phase 8 — Security Group Module

### `modules/security-group/main.tf`

```hcl
variable "vpc_id" {}

# WEB SECURITY GROUP
resource "aws_security_group" "web_sg" {
  name   = "web-sg"
  vpc_id = var.vpc_id

  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

# APP SECURITY GROUP
resource "aws_security_group" "app_sg" {
  name   = "app-sg"
  vpc_id = var.vpc_id

  ingress {
    from_port       = 80
    to_port         = 80
    protocol        = "tcp"
    security_groups = [aws_security_group.web_sg.id]
  }

  ingress {
    from_port       = 22
    to_port         = 22
    protocol        = "tcp"
    security_groups = [aws_security_group.web_sg.id]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

# RDS SECURITY GROUP
resource "aws_security_group" "rds_sg" {
  name   = "rds-sg"
  vpc_id = var.vpc_id

  ingress {
    from_port       = 3306
    to_port         = 3306
    protocol        = "tcp"
    security_groups = [aws_security_group.app_sg.id]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```
![](/img/Screenshot%202026-06-05%20085810.png)

### `modules/security-group/outputs.tf`

```hcl
output "web_sg_id" {
  value = aws_security_group.web_sg.id
}

output "app_sg_id" {
  value = aws_security_group.app_sg.id
}

output "rds_sg_id" {
  value = aws_security_group.rds_sg.id
}
```
![](/img/Screenshot%202026-06-05%20085850.png)

### Root `main.tf` (Initial)

```hcl
module "vpc" {
  source = "./modules/vpc"
}

module "security_group" {
  source = "./modules/security-group"

  vpc_id = module.vpc.vpc_id
}
```
![](/img/Screenshot%202026-06-05%20085942.png)
![](/img/Screenshot%202026-06-05%20090026.png)
![](/img/Screenshot%202026-06-05%20090214.png)

---

## Phase 9 — EC2 Module

### `modules/ec2/variables.tf`

```hcl
variable "instance_name" {}
variable "ami_id" {}
variable "instance_type" {}
variable "subnet_id" {}
variable "security_group_id" {}
variable "key_name" {}

variable "associate_public_ip" {
  default = false
}
```
![](/img/Screenshot%202026-06-05%20090434.png)

### `modules/ec2/main.tf`

```hcl
resource "aws_instance" "server" {
  ami                         = var.ami_id
  instance_type               = var.instance_type
  subnet_id                   = var.subnet_id
  vpc_security_group_ids      = [var.security_group_id]
  key_name                    = var.key_name
  associate_public_ip_address = var.associate_public_ip

  tags = {
    Name = var.instance_name
  }
}
```
![](/img/Screenshot%202026-06-05%20090509.png)

### `modules/ec2/outputs.tf`

```hcl
output "instance_id" {
  value = aws_instance.server.id
}

output "private_ip" {
  value = aws_instance.server.private_ip
}

output "public_ip" {
  value = aws_instance.server.public_ip
}
```
![](/img/Screenshot%202026-06-05%20090537.png)

---

## Phase 10 — Create Web Server

Get the latest Ubuntu 24.04 AMI ID:

```bash
aws ssm get-parameters \
  --names /aws/service/canonical/ubuntu/server/24.04/stable/current/amd64/hvm/ebs-gp3/ami-id \
  --query "Parameters[0].Value"
```

Add to root `main.tf`:

```hcl
module "web_server" {
  source = "./modules/ec2"

  instance_name       = "Web-Server"
  ami_id              = "ami-091138d0f0d41ff90"
  instance_type       = "t3.micro"
  subnet_id           = module.vpc.public_subnet_1
  security_group_id   = module.security_group.web_sg_id
  key_name            = "3tier-key"
  associate_public_ip = true
}
```

---

## Phase 11 — Create App Server

Add to root `main.tf`:

```hcl
module "app_server" {
  source = "./modules/ec2"

  instance_name       = "App-Server"
  ami_id              = "ami-091138d0f0d41ff90"
  instance_type       = "t2.micro"
  subnet_id           = module.vpc.app_subnet_1
  security_group_id   = module.security_group.app_sg_id
  key_name            = "3tier-key"
  associate_public_ip = false
}
```

> The App Server has **no public IP** — it is only reachable from within the VPC.

---

## Phase 12 — RDS Module

### `modules/rds/variables.tf`

```hcl
variable "db_name" {}
variable "db_username" {}
variable "db_password" {}
variable "subnet1" {}
variable "subnet2" {}
variable "rds_sg" {}
```
![](/img/Screenshot%202026-06-05%20091259.png)

### `modules/rds/main.tf`

```hcl
resource "aws_db_subnet_group" "db_subnet_group" {
  name = "registration-db-subnet"

  subnet_ids = [
    var.subnet1,
    var.subnet2
  ]
}

resource "aws_db_instance" "mysql" {
  identifier        = "registration-db-aftab"
  allocated_storage = 20
  engine            = "mysql"
  engine_version    = "8.0"
  instance_class    = "db.t3.micro"

  db_name  = var.db_name
  username = var.db_username
  password = var.db_password

  publicly_accessible    = false
  skip_final_snapshot    = true
  storage_type           = "gp3"

  vpc_security_group_ids = [var.rds_sg]
  db_subnet_group_name   = aws_db_subnet_group.db_subnet_group.name
}
```
![](/img/Screenshot%202026-06-05%20091443.png)
### `modules/rds/outputs.tf`

```hcl
output "rds_endpoint" {
  value = aws_db_instance.mysql.endpoint
}
```
![](/img/Screenshot%202026-06-05%20091507.png)

### Add RDS to Root `main.tf`

```hcl
module "rds" {
  source = "./modules/rds"

  db_name     = "registrationdb"
  db_username = "adminuser"
  db_password = "StrongPassword123"

  subnet1 = module.vpc.db_subnet_1
  subnet2 = module.vpc.db_subnet_2
  rds_sg  = module.security_group.rds_sg_id
}
```

---

## Phase 13 — Variables File

### `terraform.tfvars`

```hcl
db_username = "adminuser"
db_password = "StrongPassword123"
```

---

## Phase 14 — Deploy Infrastructure

### Initialize

```bash
terraform init
```
![](/img/Screenshot%202026-06-05%20104522.png)

### Validate Configuration

```bash
terraform validate
```

Expected output:
```
Success! The configuration is valid.
```

### Plan

```bash
terraform plan
```

Expected resources to be created: **25 total**, including:
- 1 VPC
- 6 Subnets
- 1 Internet Gateway
- 1 NAT Gateway
- 1 Elastic IP
- 2 Route Tables + 6 Associations
- 3 Security Groups
- 2 EC2 Instances
- 1 RDS Instance + 1 DB Subnet Group

**Terraform Plan Output (verified):**

```
Plan: 25 to add, 0 to change, 0 to destroy.

Changes to Outputs:
  app_private_ip = (known after apply)
  rds_endpoint   = (known after apply)
  web_public_ip  = (known after apply)
```
![](/img/Screenshot%202026-06-05%20104540.png)

### Apply

```bash
terraform apply
```

Type `yes` when prompted. Wait **10–15 minutes** for RDS to provision.
![](/img/Screenshot%202026-06-05%20105828.png)

---

## Phase 15 — Verify AWS Resources

After apply completes, verify in the AWS Console:

| Resource | Expected Name |
|---|---|
| VPC | `3tier-vpc` |
| Public Subnets | `Public-AZ1`, `Public-AZ2` |
| App Subnets | `App-AZ1`, `App-AZ2` |
| DB Subnets | `DB-AZ1`, `DB-AZ2` |
| EC2 Instances | `Web-Server`, `App-Server` |
| RDS Instance | `registration-db-aftab` |

![](/img/Screenshot%202026-06-05%20110022.png)
![](/img/Screenshot%202026-06-05%20110121.png)
![](/img/Screenshot%202026-06-05%20110100.png)
![](/img/Screenshot%202026-06-05%20110203.png)
---

## Phase 16 — Configure Servers Using Ansible

### Deployment Architecture at This Point

```
Internet
    │
    ▼
Web Server (Nginx)       → Public IP: 44.223.81.212
    │
    ▼
App Server (Apache+PHP)  → Private IP: 10.0.3.117
    │
    ▼
RDS MySQL                → registration-db-aftab.c6rqqyua46ii.us-east-1.rds.amazonaws.com
```

### Step 1 — Verify Ansible Installation

```bash
sudo apt update
sudo apt install ansible -y
ansible --version
```

### Step 2 — Locate and Set Private Key Permissions

```bash
find ~ -name "*.pem"
chmod 400 ~/3tier-key.pem
```

### Step 3 — Test SSH to Web Server

```bash
ssh -i ~/3tier-key.pem ubuntu@44.223.81.212
```


![](/img/Screenshot%202026-06-05%20110613.png)

### Step 4 — Test SSH to App Server (via Jump Host)

The App Server has no public IP. Connect through the Web Server as a jump host:

```bash
# From Terraform Server, SSH to Web Server
ssh -i ~/3tier-key.pem ubuntu@44.223.81.212

# Then from Web Server, SSH to App Server
ssh ubuntu@10.0.3.117
```

### Step 5 — Configure SSH ProxyJump

Create `~/.ssh/config` for seamless SSH access:

```bash
nano ~/.ssh/config
```

Paste:

```
Host web
    HostName 44.223.81.212
    User ubuntu
    IdentityFile ~/3tier-key.pem

Host app
    HostName 10.0.3.117
    User ubuntu
    IdentityFile ~/3tier-key.pem
    ProxyJump web
```

Set permissions:

```bash
chmod 600 ~/.ssh/config
```

Test direct connection to App Server:

```bash
ssh app
```

### Step 6 — Create Ansible Inventory

```bash
mkdir -p ~/terraform-3tier/ansible
cd ~/terraform-3tier/ansible
nano inventory
```

Paste:

```ini
[web]
44.223.81.212

[app]
10.0.3.117

[all:vars]
ansible_user=ubuntu
ansible_ssh_private_key_file=/home/ubuntu/3tier-key.pem

[app:vars]
ansible_ssh_common_args='-o ProxyJump=ubuntu@44.223.81.212'
```
![](/img/Screenshot%202026-06-05%20111715.png)

### Step 7 — Verify Ansible Connectivity

```bash
ansible all -i inventory -m ping
```

Expected output:

```
44.223.81.212 | SUCCESS
10.0.3.117    | SUCCESS
```
![](/img/Screenshot%202026-06-05%20111810.png)

---

## Phase 17 — Configure Ansible Inventory

If returning from another session, re-verify Ansible connectivity:

```bash
# Test Web Server
ansible web -i inventory -m ping

# Test App Server
ansible app -i inventory -m ping

# Test all
ansible all -i inventory -m ping
```
![](/img/Screenshot%202026-06-05%20112439.png)

---

## Phase 18 — Create Registration Form

### `web-files/index.html`

```bash
mkdir -p ~/terraform-3tier/web-files
nano ~/terraform-3tier/web-files/index.html
```

Paste:

```html
<!DOCTYPE html>
<html>
<head>
    <title>User Registration</title>
</head>
<body>

<h2>User Registration Form</h2>

<form action="/submit.php" method="POST">

    Name:
    <input type="text" name="name" required>
    <br><br>

    Email:
    <input type="email" name="email" required>
    <br><br>

    Phone:
    <input type="text" name="phone" required>
    <br><br>

    <input type="submit" value="Register">

</form>

</body>
</html>
```
![](/img/Screenshot%202026-06-05%20112549.png)

> **Note:** The form action uses `/submit.php` (relative path) so all traffic goes through Nginx's reverse proxy — the browser never needs to reach the private App Server directly.

### `ansible/web.yml` — Web Server Playbook

```bash
nano ~/terraform-3tier/ansible/web.yml
```

Paste:

```yaml
---
- hosts: web
  become: yes

  tasks:

    - name: Install nginx
      apt:
        name: nginx
        state: present
        update_cache: yes

    - name: Copy registration page
      copy:
        src: ../web-files/index.html
        dest: /var/www/html/index.html

    - name: Start nginx
      service:
        name: nginx
        state: started
        enabled: yes
```
![](/img/Screenshot%202026-06-05%20112644.png)

### Run Web Playbook

```bash
cd ~/terraform-3tier/ansible
ansible-playbook -i inventory web.yml
```
![](/img/Screenshot%202026-06-05%20112748.png)

Verify by opening: `http://44.223.81.212`

You should see the **User Registration Form**.

![](/img/Screenshot%202026-06-05%20112818.png)

---

## Phase 19 — Configure App Server

### Step 1 — Get RDS Endpoint

```bash
terraform output
```
![](/img/Screenshot%202026-06-05%20113101.png)

Note the RDS endpoint:

```
registration-db-aftab.c6rqqyua46ii.us-east-1.rds.amazonaws.com
```

### Step 2 — Create App Server Playbook

```bash
nano ~/terraform-3tier/ansible/app.yml
```

Paste:

```yaml
---
- hosts: app
  become: yes

  tasks:

    - name: Update packages
      apt:
        update_cache: yes

    - name: Install Apache
      apt:
        name: apache2
        state: present

    - name: Install PHP and MySQL client
      apt:
        name:
          - php
          - php-mysql
          - mysql-client
        state: present

    - name: Start Apache
      service:
        name: apache2
        state: started
        enabled: yes
```
![](/img/Screenshot%202026-06-05%20113154.png)

### Step 3 — Run App Playbook

```bash
cd ~/terraform-3tier/ansible
ansible-playbook -i inventory app.yml
```

Expected output:

```
PLAY RECAP
10.0.3.117 : ok=4  changed=4  failed=0
```
![](/img/Screenshot%202026-06-05%20113322.png)

### Step 4 — Verify Apache

```bash
ssh app
systemctl status apache2
```

Expected: `active (running)`

---

## Phase 20 — Connect App Server to RDS

### Step 1 — Test MySQL Connection from App Server

```bash
ssh app

mysql -h registration-db-aftab.c6rqqyua46ii.us-east-1.rds.amazonaws.com \
  -u adminuser \
  -p
```

Enter password: `StrongPassword123!`

You should see the `mysql>` prompt.

![](/img/Screenshot%202026-06-05%20114631.png)


### Step 2 — Create the Database Table

```sql
USE registrationdb;

CREATE TABLE users (
    id         INT AUTO_INCREMENT PRIMARY KEY,
    name       VARCHAR(100),
    email      VARCHAR(100),
    phone      VARCHAR(20),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

SHOW TABLES;
```

Expected:

```
+------------------------+
| Tables_in_registrationdb |
+------------------------+
| users                  |
+------------------------+
```

Exit MySQL:

```sql
exit;
```

---

## Phase 21 — Create submit.php

On the Terraform Server:

```bash
mkdir -p ~/terraform-3tier/app-files
nano ~/terraform-3tier/app-files/submit.php
```

Paste:

```php
<?php

$host     = "registration-db-aftab.c6rqqyua46ii.us-east-1.rds.amazonaws.com";
$username = "adminuser";
$password = "StrongPassword123!";
$database = "registrationdb";

$conn = new mysqli($host, $username, $password, $database);

if ($conn->connect_error) {
    die("Database Connection Failed: " . $conn->connect_error);
}

$name  = $_POST['name'];
$email = $_POST['email'];
$phone = $_POST['phone'];

$sql = "INSERT INTO users (name, email, phone) VALUES ('$name', '$email', '$phone')";

if ($conn->query($sql) === TRUE) {
    echo "<h2>Registration Successful</h2>";
} else {
    echo "Error: " . $conn->error;
}

$conn->close();

?>
```
![](/img/Screenshot%202026-06-05%20114825.png)

---

## Phase 22 — Deploy submit.php Using Ansible

```bash
nano ~/terraform-3tier/ansible/deploy-app.yml
```

Paste:

```yaml
---
- hosts: app
  become: yes

  tasks:

    - name: Copy submit.php to App Server
      copy:
        src: ../app-files/submit.php
        dest: /var/www/html/submit.php
        owner: root
        group: root
        mode: '0644'
```
![](/img/Screenshot%202026-06-05%20114901.png)


Run the playbook:

```bash
cd ~/terraform-3tier/ansible
ansible-playbook -i inventory deploy-app.yml
```

Expected:

```
PLAY RECAP
10.0.3.117 : ok=2  changed=1  failed=0
```
![](/img/Screenshot%202026-06-05%20114958.png)

---

## Phase 23 & 24 — Configure Nginx Reverse Proxy

The browser cannot access `10.0.3.117` directly (private subnet). Nginx on the Web Server will proxy `/submit.php` requests to the App Server.

### `web-files/default` — Nginx Configuration

```bash
nano ~/terraform-3tier/web-files/default
```

Paste:

```nginx
server {

    listen 80;
    server_name _;

    root /var/www/html;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }

    location /submit.php {
        proxy_pass http://10.0.3.117/submit.php;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```
![](/img/Screenshot%202026-06-05%20115144.png)

### `ansible/update-web.yml` — Update Web Server Playbook

```bash
nano ~/terraform-3tier/ansible/update-web.yml
```

Paste:

```yaml
---
- hosts: web
  become: yes

  tasks:

    - name: Copy updated HTML
      copy:
        src: ../web-files/index.html
        dest: /var/www/html/index.html

    - name: Copy nginx reverse proxy config
      copy:
        src: ../web-files/default
        dest: /etc/nginx/sites-available/default

    - name: Restart nginx
      service:
        name: nginx
        state: restarted
```
![](/img/Screenshot%202026-06-05%20115249.png)

### Deploy Changes

```bash
ansible-playbook -i inventory update-web.yml
```

Expected:

```
PLAY RECAP
44.223.81.212 : ok=3  changed=2  failed=0
```
![](/img/Screenshot%202026-06-05%20115336.png)
---

## Phase 25 — Final End-to-End Test

### Submit the Registration Form

Open in your browser:

```
http://44.223.81.212
```

Fill in the form:

| Field | Value |
|---|---|
| Name | Aftab |
| Email | aftab@test.com |
| Phone | 9999999999 |

Click **Register**.

![](/img/Screenshot%202026-06-05%20115442.png)

Expected response:

```
Registration Successful
```

![](/img/Screenshot%202026-06-05%20115424.png)


### Verify Data in RDS

SSH into the App Server and query the database:

```bash
ssh app

mysql -h registration-db-aftab.c6rqqyua46ii.us-east-1.rds.amazonaws.com \
  -u adminuser \
  -p
```

```sql
USE registrationdb;
SELECT * FROM users;
```
![](/img/Screenshot%202026-06-05%20115602.png)

Expected output:

```
+----+-------+----------------+------------+---------------------+
| id | name  | email          | phone      | created_at          |
+----+-------+----------------+------------+---------------------+
|  1 | Aftab | aftab@test.com | 9999999999 | 2026-xx-xx xx:xx:xx |
+----+-------+----------------+------------+---------------------+
```
![](/img/Screenshot%202026-06-05%20115615.png)
![](/img/Screenshot%202026-06-05%20120451.png)
---

## Summary of Resources Created

| Resource | Count | Details |
|---|---|---|
| VPC | 1 | `3tier-vpc` — 10.0.0.0/16 |
| Subnets | 6 | 2 Public, 2 App (Private), 2 DB (Private) |
| Internet Gateway | 1 | `3tier-igw` |
| NAT Gateway | 1 | `3tier-nat` (in Public-AZ1) |
| Elastic IP | 1 | For NAT Gateway |
| Route Tables | 2 | `Public-RT`, `Private-RT` |
| Security Groups | 3 | `web-sg`, `app-sg`, `rds-sg` |
| EC2 Instances | 2 | `Web-Server` (public), `App-Server` (private) |
| RDS MySQL | 1 | `registration-db-aftab` — db.t3.micro |

## Technology Stack

| Layer | Technology |
|---|---|
| Infrastructure as Code | Terraform 1.11.4 |
| Configuration Management | Ansible |
| Web Server | Nginx (reverse proxy) |
| App Server | Apache + PHP |
| Database | Amazon RDS MySQL 8.0 |
| Cloud Provider | AWS (us-east-1) |
| OS | Ubuntu Server 24.04 |

## Conclusion

This project successfully demonstrated the deployment of a secure and scalable 3-Tier Architecture on AWS using Terraform Modules and Ansible Automation. The complete infrastructure was provisioned through Infrastructure as Code (IaC), eliminating manual resource creation and ensuring consistency, repeatability, and faster deployments.

A custom VPC was designed with public and private subnets distributed across multiple Availability Zones. The Web Tier hosted an Nginx-based registration application in a public subnet, while the Application Tier ran Apache and PHP in a private subnet. The Database Tier consisted of an Amazon RDS MySQL instance deployed in private database subnets, ensuring that the database remained inaccessible from the public internet.

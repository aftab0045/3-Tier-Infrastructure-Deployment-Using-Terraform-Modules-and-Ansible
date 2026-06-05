# 3-Tier Infrastructure Deployment Using Terraform Modules and Ansible

## Project Overview

This project demonstrates the deployment of a complete 3-Tier Web Application Architecture on AWS using Terraform Modules and Ansible Automation.

The architecture consists of:

* Web Tier (Public Subnet)
* Application Tier (Private Subnet)
* Database Tier (Private Subnet)

The project follows Infrastructure as Code (IaC) principles using Terraform and configuration management using Ansible.

The deployed application allows users to submit registration details through a web form hosted on Nginx. The data is processed by a PHP application running on the Application Server and stored in an Amazon RDS MySQL database.

---

## Project Objectives

* Design a secure 3-Tier Architecture on AWS
* Create reusable Terraform Modules
* Automate server configuration using Ansible
* Deploy a registration form on Nginx
* Configure a PHP backend application
* Store user data in Amazon RDS MySQL
* Demonstrate secure communication between tiers
* Implement Infrastructure as Code best practices

---

# Architecture Components

## Networking Layer

### VPC

* Custom VPC
* CIDR Block: 10.0.0.0/16

### Availability Zones

* us-east-1a
* us-east-1b

### Subnets

#### Public Subnets

| Subnet     | CIDR        |
| ---------- | ----------- |
| Public-AZ1 | 10.0.1.0/24 |
| Public-AZ2 | 10.0.2.0/24 |

#### Application Subnets

| Subnet  | CIDR        |
| ------- | ----------- |
| App-AZ1 | 10.0.3.0/24 |
| App-AZ2 | 10.0.4.0/24 |

#### Database Subnets

| Subnet | CIDR        |
| ------ | ----------- |
| DB-AZ1 | 10.0.5.0/24 |
| DB-AZ2 | 10.0.6.0/24 |

### Internet Connectivity

* Internet Gateway attached to VPC
* NAT Gateway deployed in Public-AZ1
* Elastic IP associated with NAT Gateway

### Routing

#### Public Route Table

* 0.0.0.0/0 → Internet Gateway

#### Private Route Table

* 0.0.0.0/0 → NAT Gateway

---

# Security Configuration

## Web Security Group

Allowed Inbound Traffic:

* SSH (22) from Internet
* HTTP (80) from Internet
* HTTPS (443) from Internet

Outbound:

* All Traffic

## Application Security Group

Allowed Inbound Traffic:

* HTTP (80) from Web Security Group
* SSH (22) from Web Security Group

Outbound:

* All Traffic

## RDS Security Group

Allowed Inbound Traffic:

* MySQL (3306) from Application Security Group

Outbound:

* All Traffic

---

# Infrastructure Components

## Web Tier

### EC2 Instance

| Property      | Value      |
| ------------- | ---------- |
| Name          | Web-Server |
| Instance Type | t3.micro   |
| OS            | Ubuntu     |
| Subnet        | Public-AZ1 |
| Public IP     | Enabled    |

### Installed Components

* Nginx
* Registration Form
* Reverse Proxy Configuration

---

## Application Tier

### EC2 Instance

| Property      | Value      |
| ------------- | ---------- |
| Name          | App-Server |
| Instance Type | t2.micro   |
| OS            | Ubuntu     |
| Subnet        | App-AZ1    |
| Public IP     | Disabled   |

### Installed Components

* Apache2
* PHP
* PHP MySQL Module
* MySQL Client

### Application

submit.php

Responsibilities:

* Receive form submissions
* Connect to Amazon RDS
* Insert records into MySQL database

---

## Database Tier

### Amazon RDS MySQL

| Property      | Value          |
| ------------- | -------------- |
| Engine        | MySQL 8.0      |
| Instance Type | db.t3.micro    |
| Database Name | registrationdb |
| Public Access | Disabled       |

### Database Table

users

Columns:

* id
* name
* email
* phone
* created_at

---

# Terraform Module Structure

```text
terraform-3tier/
│
├── modules/
│   ├── vpc/
│   ├── security-group/
│   ├── ec2/
│   └── rds/
│
├── ansible/
│   ├── inventory
│   ├── web.yml
│   ├── app.yml
│   ├── deploy-app.yml
│   └── update-web.yml
│
├── web-files/
│   ├── index.html
│   └── default
│
├── app-files/
│   └── submit.php
│
├── provider.tf
├── variables.tf
├── terraform.tfvars
├── outputs.tf
└── main.tf
```

---

# What I Implemented

## Phase 1 – Terraform Project Setup

Created Terraform project structure and module directories.

## Phase 2 – AWS Provider Configuration

Configured:

* AWS Provider
* Terraform Version
* AWS Region (us-east-1)

## Phase 3 – VPC Creation

Created:

* Custom VPC
* DNS Hostnames
* DNS Support

## Phase 4 – Public Subnets

Created:

* Public-AZ1
* Public-AZ2

## Phase 5 – Application Subnets

Created:

* App-AZ1
* App-AZ2

## Phase 6 – Database Subnets

Created:

* DB-AZ1
* DB-AZ2

## Phase 7 – Internet Gateway

Attached Internet Gateway to VPC.

## Phase 8 – NAT Gateway

Created:

* Elastic IP
* NAT Gateway

Configured outbound internet access for private resources.

## Phase 9 – Route Tables

Configured:

* Public Route Table
* Private Route Table

Associated all subnets.

## Phase 10 – Security Groups

Created:

* Web Security Group
* App Security Group
* RDS Security Group

Implemented least-privilege access.

## Phase 11 – EC2 Deployment

Created:

* Web Server
* Application Server

Using reusable Terraform EC2 module.

## Phase 12 – RDS Deployment

Created:

* DB Subnet Group
* MySQL RDS Instance

## Phase 13 – Terraform Deployment

Executed:

```bash
terraform init
terraform plan
terraform apply
```

Successfully provisioned all AWS resources.

## Phase 14 – SSH Connectivity

Configured:

* SSH Key Pair
* Bastion-style access through Web Server
* SSH access to private App Server

## Phase 15 – Ansible Setup

Installed:

* Ansible
* Inventory configuration

Verified connectivity using:

```bash
ansible all -m ping
```

## Phase 16 – Web Server Configuration

Automated installation of:

* Nginx

Deployed:

* Registration Form

## Phase 17 – Application Server Configuration

Automated installation of:

* Apache2
* PHP
* PHP-MySQL
* MySQL Client

## Phase 18 – Database Connectivity

Connected Application Server to RDS MySQL.

## Phase 19 – Database Setup

Created:

```sql
CREATE TABLE users (
id INT AUTO_INCREMENT PRIMARY KEY,
name VARCHAR(100),
email VARCHAR(100),
phone VARCHAR(20),
created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

## Phase 20 – PHP Application Deployment

Created:

submit.php

Features:

* Connects to RDS
* Receives form data
* Inserts records into database

## Phase 21 – Reverse Proxy Configuration

Configured Nginx Reverse Proxy:

```text
Browser
   ↓
Web Server (Nginx)
   ↓
App Server (Apache + PHP)
   ↓
Amazon RDS MySQL
```

## Phase 22 – End-to-End Testing

Verified:

* Form submission
* Database insertion
* Record retrieval

## Phase 23 – Terraform Outputs

Configured outputs:

* Web Public IP
* App Private IP
* RDS Endpoint

## Phase 24 – Validation

Verified:

* Web Server Status
* App Server Status
* RDS Connectivity
* Security Groups
* Database Operations

## Phase 25 – Final Verification

Successfully stored registration data in RDS:

```sql
SELECT * FROM users;
```

Result:

```text
Aftab Attar
aftab@gmail.com
9999999999
```

Project completed successfully.

---

# Deployment Steps

## Initialize Terraform

```bash
terraform init
```

## Review Infrastructure

```bash
terraform plan
```

## Deploy Infrastructure

```bash
terraform apply
```

## Verify Outputs

```bash
terraform output
```

## Configure Servers

```bash
ansible-playbook -i inventory web.yml

ansible-playbook -i inventory app.yml

ansible-playbook -i inventory deploy-app.yml

ansible-playbook -i inventory update-web.yml
```

---

# Testing

Open:

```text
http://<WEB_PUBLIC_IP>
```

Fill:

* Name
* Email
* Phone

Click:

Register

Expected Result:

```text
Registration Successful
```

Verify data:

```sql
SELECT * FROM users;
```

---

# Key Learning Outcomes

* Terraform Modules
* Infrastructure as Code
* AWS Networking
* EC2 Automation
* Amazon RDS
* Security Groups
* NAT Gateway
* Ansible Automation
* Nginx Reverse Proxy
* PHP MySQL Integration
* Multi-Tier Architecture Design
* Cloud Infrastructure Deployment


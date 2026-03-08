# Day-24 Lab

## Topic: Networking Fundamentals, AWS VPC Architecture, and Private Subnet Access

---

#  Objective

In this lab we explored **networking from basic real-world concepts to cloud networking architecture in AWS**.

The lab covers:

* How home and internet networks work
* Networking devices such as **Router, Switch, Hub, Modem, ISP**
* Cloud networking components like **VPC, Subnets, Security Groups, NAT, IGW**
* Designing **Public and Private Subnet architecture**
* Accessing a **Private VM through a Public VM (Jump Host)**
* Installing **Nginx/Apache web server inside private VM**

---

# Basic Networking (Real World)

Before understanding cloud networking, it is important to understand how networking works in a **home or office environment**.

Example:

```
Laptop / Mobile
        │
      WiFi
        │
      Router
        │
      Modem
        │
        ISP
        │
     Internet
```

---

#  Networking Devices

---

# Hub

A **Hub** is the simplest networking device.

Characteristics:

* Broadcasts data to **all connected devices**
* No intelligence
* Operates at **Layer 1 (Physical Layer)**

Example:

```
Device A → Hub → Device B, C, D (all receive data)
```

---

# Switch

A **Switch** connects devices within the same network.

Characteristics:

* Sends data only to the intended device
* Uses **MAC address table**
* Operates at **Layer 2 (Data Link Layer)**

Example:

```
Laptop → Switch → Printer
```

---

# Router

A **Router** connects different networks together.

Characteristics:

* Routes traffic between networks
* Uses **IP addresses**
* Operates at **Layer 3 (Network Layer)**

Example:

```
Home Network → Router → Internet
```

---

# Modem

A **Modem** connects your local network to the ISP.

Purpose:

* Converts digital signals to analog signals
* Allows internet connectivity

Example:

```
Router → Modem → ISP
```

---

# ISP (Internet Service Provider)

An **ISP** provides internet connectivity to users.

Examples:

* Airtel
* Jio
* Comcast
* Verizon

The ISP connects your network to the **global internet infrastructure**.

---

# 📱 Device Connectivity Examples

---

# Laptop Using WiFi

```
Laptop
   │
WiFi Router
   │
Modem
   │
ISP
   │
Internet
```

---

# Mobile Using Mobile Data

```
Mobile Device
      │
Mobile Tower
      │
ISP Network
      │
Internet
```

---

#  AWS Cloud Networking

In AWS, networking is implemented using **Virtual Private Cloud (VPC)**.

---

# VPC (Virtual Private Cloud)

A **VPC** is a logically isolated virtual network in AWS.

Features:

* Define IP range
* Create subnets
* Configure routing
* Control security

Example:

```
VPC: 10.0.0.0/16
```

---

# Subnet

A **Subnet** is a smaller network inside a VPC.

Types:

| Subnet Type    | Purpose                       |
| -------------- | ----------------------------- |
| Public Subnet  | Internet accessible resources |
| Private Subnet | Internal services             |

Example:

```
VPC
│
├── Public Subnet
└── Private Subnet
```

---

# Internet Gateway (IGW)

An **Internet Gateway** enables internet access for resources inside a VPC.

Functions:

* Allows inbound internet traffic
* Allows outbound internet traffic

Used with **Public Subnets**.

---

# NAT Gateway

A **NAT Gateway** allows private subnet resources to access the internet **without exposing them publicly**.

Example:

```
Private VM → NAT → Internet
```

---

# Route Table

A **Route Table** determines how traffic flows inside the VPC.

Example:

| Destination | Target           |
| ----------- | ---------------- |
| 0.0.0.0/0   | Internet Gateway |
| 10.0.0.0/16 | Local            |

Each subnet must be associated with a route table.

---

# Security Groups (SG)

Security Groups act as **virtual firewalls for EC2 instances**.

Characteristics:

* Stateful
* Allow rules only

Example rules:

| Type | Port |
| ---- | ---- |
| SSH  | 22   |
| HTTP | 80   |

---

# Network ACL (NACL)

A **Network ACL** is a subnet-level firewall.

Characteristics:

* Stateless
* Allow and deny rules

Controls traffic entering and leaving subnets.

---

# VPC Peering

VPC Peering allows **communication between two VPC networks**.

Example:

```
VPC-A ←→ VPC-B
```

Used for connecting networks across accounts or environments.

---

# Elastic IP

An **Elastic IP** is a static public IP address.

Characteristics:

* Persistent public IP
* Can be attached to EC2 instances
* Useful for public services

---

# VPC Flow Logs

VPC Flow Logs capture **network traffic information**.

They help with:

* Monitoring traffic
* Troubleshooting connectivity
* Security auditing

Example data captured:

* Source IP
* Destination IP
* Port
* Protocol
* Accept / Reject status

---

# Task-1: Create Public and Private Subnet Architecture

---

# Architecture Overview

```
                Internet
                    │
                Internet Gateway
                    │
               Public Subnet
                    │
                Public VM
                    │
             Private Subnet
                    │
                Private VM
```

---

# Step-1: Create VPC

Example CIDR:

```
10.0.0.0/16
```

---

# Step-2: Create Two Subnets

| Subnet         | CIDR        |
| -------------- | ----------- |
| Public Subnet  | 10.0.1.0/24 |
| Private Subnet | 10.0.2.0/24 |

---

# Step-3: Create Internet Gateway

Attach the IGW to the VPC.

---

# Step-4: Create Route Tables

Two route tables are created:

### Public Route Table

| Destination | Target           |
| ----------- | ---------------- |
| 0.0.0.0/0   | Internet Gateway |

Associated with **Public Subnet**.

---

### Private Route Table

| Destination | Target |
| ----------- | ------ |
| 10.0.0.0/16 | Local  |

Associated with **Private Subnet**.

---

# Step-5: Launch EC2 Instances

| Instance   | Subnet         |
| ---------- | -------------- |
| Public VM  | Public Subnet  |
| Private VM | Private Subnet |

Public VM receives **Public IP**.

Private VM receives **Private IP only**.

---

# Step-6: Access Private VM

Access flow:

```
Laptop → Public VM (SSH)
              │
              │ SSH
              ▼
         Private VM
```

Public VM acts as a **Jump Host (Bastion Host)**.

---

# Task-2: Install Web Server in Private VM

After connecting to Private VM:

Install **Nginx or Apache**.

## Create the NAT gateway

Go to the VPC section and click on NAT gateway and create a new NAT gateway

---
## Attach it to the Route Table

Go to the VPC section and click on Route Tables and select the private route table and click on edit route and add a new route as the NAT gateway.

---

## Install Nginx

```bash
sudo apt update
sudo apt install nginx -y
```

---

## Install Apache

```bash
sudo apt update
sudo apt install apache2 -y
```

---

# Verify Web Server

Check service status:

```bash
sudo systemctl status nginx
```

or

```bash
sudo systemctl status apache2
```

---

# Test Web Server

From Public VM:

```bash
curl <private-vm-ip>
```

Example:

```
curl 10.0.2.25
```

Expected output:

```
Nginx Welcome Page
```

---
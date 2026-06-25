**LOW-LEVEL DESIGN DOCUMENT**

AWS VPC Architecture & VPC Peering

XYZ Corporation - Module 5 Case Study

| **Project**       | VPC & Peering Infrastructure Design     |
| ----------------- | --------------------------------------- |
| **Author**        | Rahul Kumar Pandey                      |
| **Contact**       | <rahulpandeyyy@gmail.com> \| 7257000333 |
| **Document Type** | Low-Level Design (LLD)                  |
| **Version**       | 1.0                                     |
| **Date**          | June 2026                               |
| **Status**        | Final                                   |

**CONFIDENTIAL - INTERNAL USE ONLY**

# **1\. Introduction**

## **1.1 Purpose**

This Low-Level Design (LLD) document provides the detailed technical specifications for deploying a multi-environment AWS VPC infrastructure for XYZ Corporation. It covers the complete network topology, subnet design, routing configuration, security controls, and inter-VPC connectivity using VPC Peering.

## **1.2 Scope**

- Design and provisioning of Production VPC (4-tier architecture)
- Design and provisioning of Development VPC (2-tier architecture)
- VPC Peering configuration between Production and Development VPCs
- Security Group and NACL definitions for each tier
- Routing table configurations for internet and inter-VPC traffic
- NAT Gateway setup for outbound internet access from private subnets

## **1.3 Audience**

- Cloud Infrastructure Engineers
- Network Architects
- DevOps / SRE Teams
- Security Review Team

## **1.4 Document Conventions**

- CIDR blocks follow RFC 1918 private address ranges
- All resource names follow snake_case or kebab-case conventions
- Security groups use least-privilege principle
- NACLs provide subnet-level defense in depth

# **2\. Architecture Overview**

## **2.1 High-Level Architecture Summary**

The solution comprises two logically isolated VPCs connected via AWS VPC Peering:

| **Environment**               | **CIDR Block** | **Architecture**                         |
| ----------------------------- | -------------- | ---------------------------------------- |
| **Production VPC (Prod-VPC)** | 10.0.0.0/16    | 4-Tier: Web → App1 → App2 → DBCache → DB |
| **Development VPC (Dev-VPC)** | 10.1.0.0/16    | 2-Tier: Web → DB                         |
| **VPC Peering**               | Prod ↔ Dev     | DB subnet communication across VPCs      |

## **2.2 Network Topology Diagram (Textual)**

The following ASCII-style block represents the logical network topology:

┌──────────────────────────── PROD-VPC (10.0.0.0/16) ─────────────────────────┐

│ ┌───────────────────────────────────────────────────────────────────────┐ │

│ │ \[Internet Gateway\] \[NAT Gateway\] │ │

│ │ │ │ │ │

│ │ PUBLIC SUBNET: web (10.0.1.0/24) ─── EC2: web │ │

│ │ │ │ │ │

│ │ PRIVATE: app1 (10.0.2.0/24) ────── EC2: app1 ← NAT route │ │

│ │ PRIVATE: app2 (10.0.3.0/24) ────── EC2: app2 │ │

│ │ PRIVATE: dbcache (10.0.4.0/24) ─── EC2: dbcache ← NAT route │ │

│ │ PRIVATE: db (10.0.5.0/24) ──────── EC2: db │ │

│ └───────────────────────────────────────────────────────────────────────┘ │

│ │ VPC Peering (db↔db) │

└─────────────────────────────────────────┼────────────────────────────────────┘

│

┌─────────────────────────────────────────┼──── DEV-VPC (10.1.0.0/16) ─────────┐

│ ┌──────────────────────────────────────┼──────────────────────────────────┐ │

│ │ \[Internet Gateway\] │ │ │

│ │ │ │ │ │

│ │ PUBLIC: web (10.1.1.0/24) ─── EC2: web │ │

│ │ PRIVATE: db (10.1.2.0/24) ─── EC2: db ← Peering to Prod DB │ │

│ └─────────────────────────────────────────────────────────────────────── ┘ │

└───────────────────────────────────────────────────────────────────────────────┘

# **3\. Production VPC - Detailed Design**

## **3.1 VPC Configuration**

| **Parameter**        | **Value**                                            |
| -------------------- | ---------------------------------------------------- |
| **VPC Name**         | Prod-VPC                                             |
| **CIDR Block**       | 10.0.0.0/16                                          |
| **Region**           | ap-south-1 (Mumbai)                                  |
| **Tenancy**          | Default                                              |
| **DNS Hostnames**    | Enabled                                              |
| **DNS Resolution**   | Enabled                                              |
| **Internet Gateway** | Prod-IGW (attached to Prod-VPC)                      |
| **NAT Gateway**      | Prod-NAT-GW (deployed in web subnet with Elastic IP) |

## **3.2 Subnet Design**

The Production VPC is divided into 5 subnets across a 4-tier architecture:

| **Subnet Name** | **CIDR Block** | **Type** | **Tier** | **AZ**      | **EC2 Instance** |
| --------------- | -------------- | -------- | -------- | ----------- | ---------------- |
| **web**         | 10.0.1.0/24    | Public   | Tier 1   | ap-south-1a | web              |
| **app1**        | 10.0.2.0/24    | Private  | Tier 2   | ap-south-1a | app1             |
| **app2**        | 10.0.3.0/24    | Private  | Tier 2   | ap-south-1b | app2             |
| **dbcache**     | 10.0.4.0/24    | Private  | Tier 3   | ap-south-1a | dbcache          |
| **db**          | 10.0.5.0/24    | Private  | Tier 4   | ap-south-1b | db               |

## **3.3 Route Tables**

### **3.3.1 Public Route Table (web subnet)**

| **Destination** | **Target**   | **Description**         |
| --------------- | ------------ | ----------------------- |
| 10.0.0.0/16     | local        | Local VPC traffic       |
| 0.0.0.0/0       | Prod-IGW     | Internet access via IGW |
| 10.1.0.0/16     | pcx-xxxxxxxx | Dev VPC via peering     |

### **3.3.2 Private Route Table - app1 & dbcache (NAT-enabled)**

| **Destination** | **Target**   | **Description**           |
| --------------- | ------------ | ------------------------- |
| 10.0.0.0/16     | local        | Local VPC traffic         |
| 0.0.0.0/0       | Prod-NAT-GW  | Outbound internet via NAT |
| 10.1.0.0/16     | pcx-xxxxxxxx | Dev VPC via peering       |

### **3.3.3 Private Route Table - app2 & db (no internet)**

| **Destination** | **Target**   | **Description**        |
| --------------- | ------------ | ---------------------- |
| 10.0.0.0/16     | local        | Local VPC traffic only |
| 10.1.0.0/16     | pcx-xxxxxxxx | Dev VPC via peering    |

# **4\. Security Configuration - Production VPC**

## **4.1 Security Groups**

### **4.1.1 Web Security Group (SG-Web)**

| **Direction** | **Protocol** | **Port** | **Source / Destination** | **Purpose**          |
| ------------- | ------------ | -------- | ------------------------ | -------------------- |
| Inbound       | TCP          | 80       | 0.0.0.0/0                | HTTP from internet   |
| Inbound       | TCP          | 443      | 0.0.0.0/0                | HTTPS from internet  |
| Inbound       | TCP          | 22       | Admin IP / Bastion       | SSH management       |
| Outbound      | All          | All      | 0.0.0.0/0                | All outbound allowed |

### **4.1.2 Application Security Group (SG-App - applies to app1, app2)**

| **Direction** | **Protocol** | **Port** | **Source / Destination** | **Purpose**                  |
| ------------- | ------------ | -------- | ------------------------ | ---------------------------- |
| Inbound       | TCP          | 8080     | SG-Web                   | App traffic from Web tier    |
| Inbound       | TCP          | 22       | SG-Web / Bastion         | SSH from web/bastion         |
| Outbound      | TCP          | 6379     | SG-DBCache               | Redis/Cache traffic          |
| Outbound      | All          | All      | 0.0.0.0/0                | Internet via NAT (app1 only) |

### **4.1.3 DBCache Security Group (SG-DBCache)**

| **Direction** | **Protocol** | **Port** | **Source / Destination** | **Purpose**             |
| ------------- | ------------ | -------- | ------------------------ | ----------------------- |
| Inbound       | TCP          | 6379     | SG-App                   | Redis from App tier     |
| Inbound       | TCP          | 11211    | SG-App                   | Memcached from App tier |
| Outbound      | TCP          | 3306     | SG-DB                    | MySQL to DB tier        |
| Outbound      | All          | All      | 0.0.0.0/0                | Internet via NAT        |

### **4.1.4 DB Security Group (SG-DB - Production)**

| **Direction** | **Protocol** | **Port** | **Source / Destination** | **Purpose**                    |
| ------------- | ------------ | -------- | ------------------------ | ------------------------------ |
| Inbound       | TCP          | 3306     | SG-DBCache               | MySQL from cache tier          |
| Inbound       | TCP          | 3306     | 10.1.2.0/24 (Dev DB)     | MySQL from Dev VPC via peering |
| Outbound      | All          | All      | VPC CIDR                 | Internal traffic only          |

## **4.2 Network ACLs (NACLs)**

NACLs provide a stateless second layer of defense at the subnet boundary. Rules are evaluated in ascending order; first match wins.

### **4.2.1 Public Subnet NACL (web)**

| **Rule #** | **Direction** | **Protocol** | **Port**   | **Source/Dest** | **Action** | **Notes**    |
| ---------- | ------------- | ------------ | ---------- | --------------- | ---------- | ------------ |
| 100        | Inbound       | TCP          | 80         | 0.0.0.0/0       | ALLOW      | HTTP         |
| 110        | Inbound       | TCP          | 443        | 0.0.0.0/0       | ALLOW      | HTTPS        |
| 120        | Inbound       | TCP          | 1024-65535 | 0.0.0.0/0       | ALLOW      | Ephemeral    |
| \*         | Inbound       | All          | All        | 0.0.0.0/0       | DENY       | Default deny |
| 100        | Outbound      | TCP          | All        | 0.0.0.0/0       | ALLOW      | All out      |
| \*         | Outbound      | All          | All        | 0.0.0.0/0       | DENY       | Default deny |

### **4.2.2 Private Subnet NACLs (app1, app2, dbcache, db)**

| **Rule #** | **Direction** | **Protocol** | **Port**   | **Source/Dest** | **Action** | **Notes**       |
| ---------- | ------------- | ------------ | ---------- | --------------- | ---------- | --------------- |
| 100        | Inbound       | TCP          | App port   | 10.0.0.0/16     | ALLOW      | VPC internal    |
| 110        | Inbound       | TCP          | 1024-65535 | 0.0.0.0/0       | ALLOW      | Ephemeral       |
| 120        | Inbound       | TCP          | All        | 10.1.0.0/16     | ALLOW      | Dev VPC peering |
| \*         | Inbound       | All          | All        | 0.0.0.0/0       | DENY       | Default deny    |
| 100        | Outbound      | TCP          | All        | 10.0.0.0/16     | ALLOW      | VPC traffic     |
| 110        | Outbound      | TCP          | All        | 10.1.0.0/16     | ALLOW      | Dev VPC peering |
| \*         | Outbound      | All          | All        | 0.0.0.0/0       | DENY       | Default deny    |

# **5\. Development VPC - Detailed Design**

## **5.1 VPC Configuration**

| **Parameter**        | **Value**                                       |
| -------------------- | ----------------------------------------------- |
| **VPC Name**         | Dev-VPC                                         |
| **CIDR Block**       | 10.1.0.0/16                                     |
| **Region**           | ap-south-1 (Mumbai)                             |
| **Tenancy**          | Default                                         |
| **Internet Gateway** | Dev-IGW (attached to Dev-VPC)                   |
| **NAT Gateway**      | Not required (db subnet has no internet access) |

## **5.2 Subnet Design**

| **Subnet Name** | **CIDR Block** | **Type** | **Tier** | **AZ**      | **EC2 Instance** |
| --------------- | -------------- | -------- | -------- | ----------- | ---------------- |
| **web**         | 10.1.1.0/24    | Public   | Tier 1   | ap-south-1a | web              |
| **db**          | 10.1.2.0/24    | Private  | Tier 2   | ap-south-1a | db               |

## **5.3 Route Tables**

### **5.3.1 Public Route Table (web subnet)**

| **Destination** | **Target**   | **Description**         |
| --------------- | ------------ | ----------------------- |
| 10.1.0.0/16     | local        | Local VPC traffic       |
| 0.0.0.0/0       | Dev-IGW      | Internet access via IGW |
| 10.0.0.0/16     | pcx-xxxxxxxx | Prod VPC via peering    |

### **5.3.2 Private Route Table (db subnet)**

| **Destination** | **Target**   | **Description**           |
| --------------- | ------------ | ------------------------- |
| 10.1.0.0/16     | local        | Local VPC traffic only    |
| 10.0.0.0/16     | pcx-xxxxxxxx | Prod VPC DB communication |

## **5.4 Security Groups - Development VPC**

### **5.4.1 Dev Web Security Group (SG-Dev-Web)**

| **Direction** | **Protocol** | **Port** | **Source/Dest** | **Purpose**         |
| ------------- | ------------ | -------- | --------------- | ------------------- |
| Inbound       | TCP          | 80       | 0.0.0.0/0       | HTTP from internet  |
| Inbound       | TCP          | 443      | 0.0.0.0/0       | HTTPS from internet |
| Inbound       | TCP          | 22       | Admin IP        | SSH management      |
| Outbound      | All          | All      | 0.0.0.0/0       | All outbound        |

### **5.4.2 Dev DB Security Group (SG-Dev-DB)**

| **Direction** | **Protocol** | **Port** | **Source/Dest**       | **Purpose**                    |
| ------------- | ------------ | -------- | --------------------- | ------------------------------ |
| Inbound       | TCP          | 3306     | SG-Dev-Web            | MySQL from Dev Web tier        |
| Inbound       | TCP          | 3306     | 10.0.5.0/24 (Prod DB) | MySQL from Prod DB via peering |
| Outbound      | All          | All      | 10.1.0.0/16           | Internal VPC only              |

# **6\. VPC Peering Configuration**

## **6.1 Peering Overview**

| **Parameter**               | **Value**                          |
| --------------------------- | ---------------------------------- |
| **Peering Connection Name** | Prod-Dev-Peering                   |
| **Requester VPC**           | Prod-VPC (10.0.0.0/16)             |
| **Accepter VPC**            | Dev-VPC (10.1.0.0/16)              |
| **Region**                  | Same region (intra-region peering) |
| **Status**                  | Active (accepted)                  |
| **DNS Resolution**          | Enabled on both sides              |
| **CIDR Overlap**            | None (10.0.0.0/16 vs 10.1.0.0/16)  |

## **6.2 Route Table Updates for Peering**

Both VPCs must have updated route tables to allow traffic through the peering connection:

| **Route Table**     | **Destination** | **Target**   | **Subnet Association** |
| ------------------- | --------------- | ------------ | ---------------------- |
| Prod-Public-RT      | 10.1.0.0/16     | pcx-xxxxxxxx | web                    |
| Prod-Private-NAT-RT | 10.1.0.0/16     | pcx-xxxxxxxx | app1, dbcache          |
| Prod-Private-RT     | 10.1.0.0/16     | pcx-xxxxxxxx | app2, db               |
| Dev-Public-RT       | 10.0.0.0/16     | pcx-xxxxxxxx | web                    |
| Dev-Private-RT      | 10.0.0.0/16     | pcx-xxxxxxxx | db                     |

## **6.3 DB-to-DB Communication Flow**

The peering connection enables direct private communication between the database subnets of both VPCs. No NAT or internet routing is required.

Prod-DB (10.0.5.x) ─────\[pcx-xxxxxxxx\]────► Dev-DB (10.1.2.x)

│ │

SG-DB (Inbound 3306 from 10.1.2.0/24) SG-Dev-DB (Inbound 3306 from 10.0.5.0/24)

NACL allows 10.1.0.0/16 NACL allows 10.0.0.0/16

## **6.4 Peering Limitations & Notes**

- VPC Peering is non-transitive: traffic cannot pass through a peering connection to reach a third VPC
- Overlapping CIDR blocks are not supported; 10.0.0.0/16 and 10.1.0.0/16 have no overlap
- Security Groups can reference peered-VPC SGs only in the same region
- VPC Peering does not support edge-to-edge routing (e.g., through VPN or Direct Connect)

# **7\. EC2 Instance Specifications**

## **7.1 Production VPC Instances**

| **Instance Name** | **Subnet**        | **Instance Type** | **AMI**        | **Security Group** | **Public IP** |
| ----------------- | ----------------- | ----------------- | -------------- | ------------------ | ------------- |
| **web**           | web (Public)      | t3.micro          | Amazon Linux 2 | SG-Web             | Yes (EIP)     |
| **app1**          | app1 (Private)    | t3.small          | Amazon Linux 2 | SG-App             | No            |
| **app2**          | app2 (Private)    | t3.small          | Amazon Linux 2 | SG-App             | No            |
| **dbcache**       | dbcache (Private) | r5.large          | Amazon Linux 2 | SG-DBCache         | No            |
| **db**            | db (Private)      | r5.xlarge         | Amazon Linux 2 | SG-DB              | No            |

## **7.2 Development VPC Instances**

| **Instance Name** | **Subnet**   | **Instance Type** | **AMI**        | **Security Group** | **Public IP** |
| ----------------- | ------------ | ----------------- | -------------- | ------------------ | ------------- |
| **web**           | web (Public) | t3.micro          | Amazon Linux 2 | SG-Dev-Web         | Yes (EIP)     |
| **db**            | db (Private) | t3.medium         | Amazon Linux 2 | SG-Dev-DB          | No            |

# **8\. Implementation Steps**

## **8.1 Production VPC Deployment Sequence**

- Create Prod-VPC with CIDR 10.0.0.0/16
- Create Internet Gateway (Prod-IGW) and attach to Prod-VPC
- Create 5 subnets: web (10.0.1.0/24), app1 (10.0.2.0/24), app2 (10.0.3.0/24), dbcache (10.0.4.0/24), db (10.0.5.0/24)
- Allocate Elastic IP for NAT Gateway
- Create NAT Gateway (Prod-NAT-GW) in web subnet using the Elastic IP
- Create and configure route tables: Public-RT, Private-NAT-RT, Private-RT
- Associate subnets with respective route tables
- Create Security Groups: SG-Web, SG-App, SG-DBCache, SG-DB with defined rules
- Configure NACLs for each subnet with stateless rules
- Launch EC2 instances in each subnet with the appropriate SGs and key pairs

## **8.2 Development VPC Deployment Sequence**

- Create Dev-VPC with CIDR 10.1.0.0/16
- Create Internet Gateway (Dev-IGW) and attach to Dev-VPC
- Create 2 subnets: web (10.1.1.0/24), db (10.1.2.0/24)
- Create route tables: Dev-Public-RT, Dev-Private-RT
- Associate subnets: web → Dev-Public-RT, db → Dev-Private-RT
- Create Security Groups: SG-Dev-Web, SG-Dev-DB
- Configure NACLs for both subnets
- Launch EC2 instances: web (public subnet), db (private subnet)

## **8.3 VPC Peering Setup**

- From Prod-VPC, initiate a VPC Peering Connection request to Dev-VPC
- Accept the peering connection from Dev-VPC
- Update all Prod-VPC route tables to add destination 10.1.0.0/16 → pcx-id
- Update all Dev-VPC route tables to add destination 10.0.0.0/16 → pcx-id
- Update SG-DB inbound rules to allow port 3306 from 10.1.2.0/24
- Update SG-Dev-DB inbound rules to allow port 3306 from 10.0.5.0/24
- Verify connectivity: from Prod-DB, ping / connect to Dev-DB private IP and vice versa

# **9\. Validation & Testing Checklist**

| **Test Case**                                             | **Expected Result** | **Status**      |
| --------------------------------------------------------- | ------------------- | --------------- |
| Prod web instance accessible via HTTP/HTTPS from internet | Success (200 OK)    | ☐ Pass / ☐ Fail |
| Prod app1 can reach internet via NAT Gateway              | curl succeeds       | ☐ Pass / ☐ Fail |
| Prod dbcache can reach internet via NAT Gateway           | curl succeeds       | ☐ Pass / ☐ Fail |
| Prod app2 cannot reach internet (no route)                | Connection timeout  | ☐ Pass / ☐ Fail |
| Prod db cannot reach internet                             | Connection timeout  | ☐ Pass / ☐ Fail |
| Dev web instance accessible via HTTP/HTTPS from internet  | Success (200 OK)    | ☐ Pass / ☐ Fail |
| Dev db cannot reach internet                              | Connection timeout  | ☐ Pass / ☐ Fail |
| Prod-DB can connect to Dev-DB on port 3306                | MySQL connect OK    | ☐ Pass / ☐ Fail |
| Dev-DB can connect to Prod-DB on port 3306                | MySQL connect OK    | ☐ Pass / ☐ Fail |
| VPC Peering traffic is NOT routed to internet             | Private IP routing  | ☐ Pass / ☐ Fail |
| SG deny rules block unauthorized cross-tier traffic       | Connection refused  | ☐ Pass / ☐ Fail |

# **10\. IP Addressing Summary**

## **10.1 Complete CIDR Allocation Table**

| **Resource** | **VPC**     | **CIDR**    | **Type** | **Hosts** | **Internet** |
| ------------ | ----------- | ----------- | -------- | --------- | ------------ |
| **Prod-VPC** | Production  | 10.0.0.0/16 | VPC      | 65,536    | IGW + NAT    |
| web          | Production  | 10.0.1.0/24 | Public   | 254       | IGW          |
| app1         | Production  | 10.0.2.0/24 | Private  | 254       | NAT GW       |
| app2         | Production  | 10.0.3.0/24 | Private  | 254       | None         |
| dbcache      | Production  | 10.0.4.0/24 | Private  | 254       | NAT GW       |
| db           | Production  | 10.0.5.0/24 | Private  | 254       | None         |
| **Dev-VPC**  | Development | 10.1.0.0/16 | VPC      | 65,536    | IGW only     |
| web          | Development | 10.1.1.0/24 | Public   | 254       | IGW          |
| db           | Development | 10.1.2.0/24 | Private  | 254       | None         |

## **10.2 Key Design Decisions**

- Non-overlapping CIDRs between Prod and Dev VPCs ensure VPC Peering compatibility
- Each tier is isolated in its own /24 subnet, providing 254 usable host addresses
- Only app1 and dbcache are permitted outbound internet via NAT - all other private subnets are fully isolated
- The db subnets in both VPCs communicate exclusively via VPC Peering using private IPs - no internet exposure

# **11\. Risk & Mitigation**

| **Risk**                        | **Severity** | **Impact**                  | **Mitigation**                         |
| ------------------------------- | ------------ | --------------------------- | -------------------------------------- |
| NAT GW single point of failure  | High         | app1, dbcache lose internet | Deploy NAT GW in multiple AZs          |
| Peering connection disruption   | High         | DB sync fails               | Monitor with CloudWatch alarms         |
| Over-permissive security groups | Critical     | Lateral movement risk       | Regular SG audits, least privilege     |
| CIDR exhaustion in /24 subnets  | Medium       | Cannot add more hosts       | Reserve capacity, plan for expansion   |
| Accidental public IP assignment | High         | Private subnet exposed      | Disable auto-assign on private subnets |

**END OF DOCUMENT**

Low-Level Design - AWS VPC & Peering | XYZ Corporation | v1.0 | June 2026
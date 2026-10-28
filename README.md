# Production Three-Tier Web Application Deployment on Microsoft Azure

[![Azure](https://img.shields.io/badge/Microsoft_Azure-Cloud_Infrastructure-0078D4?logo=microsoftazure&logoColor=white)](https://azure.microsoft.com/)
[![Ubuntu](https://img.shields.io/badge/Ubuntu_24.04_LTS-Operating_System-E95420?logo=ubuntu&logoColor=white)](https://ubuntu.com/)
[![Next.js](https://img.shields.io/badge/Next.js_15-Frontend-black?logo=nextdotjs&logoColor=white)](https://nextjs.org/)
[![Node.js](https://img.shields.io/badge/Node.js_Express-Backend_API-339933?logo=nodedotjs&logoColor=white)](https://nodejs.org/)
[![MySQL](https://img.shields.io/badge/Azure_Database_for_MySQL-Flexible_Server-4479A1?logo=mysql&logoColor=white)](https://azure.microsoft.com/)
[![Nginx](https://img.shields.io/badge/Nginx-Reverse_Proxy-009639?logo=nginx&logoColor=white)](https://nginx.org/)
[![Networking](https://img.shields.io/badge/Azure_VNet-Subnet_Segmentation-blue)](https://azure.microsoft.com/)

---

## Table of Contents
1. [Executive Summary & Scenario](#1-executive-summary--scenario)
2. [End-to-End Architectural Blueprint](#2-end-to-end-architectural-blueprint)
3. [Network Segmentation & IP Addressing Scheme](#3-network-segmentation--ip-addressing-scheme)
4. [Task 1: Architecture Design & Implementation Strategy](#task-1-architecture-design--implementation-strategy)
5. [Task 2: Azure Network Foundation Deployment](#task-2-azure-network-foundation-deployment)
6. [Task 3: Security & Network Access Control](#task-3-security--network-access-control)
7. [Upcoming Implementation Phases (Tasks 4–8)](#upcoming-implementation-phases-tasks-48)
8. [Author & Engineering Profile](#author--engineering-profile)

---

## 1. Executive Summary & Scenario

### Real-World Business Context
Modern cloud architectures require strict isolation between public presentation layers, internal business logic, and backend databases. In this enterprise project, the **Book Review Application** is migrated and deployed into a production-grade, highly secure, three-tier environment on **Microsoft Azure**.

The deployment enforces zero direct public ingress to backend compute and database tiers, implements managed egress routing via Azure NAT Gateway, provides resilient layer-4 traffic distribution through dual Azure Load Balancers (Public and Internal), and enforces end-to-end least-privilege access control.

### Key Engineering Objectives
* **Logical & Network Segmentation:** Isolate Presentation, Application, and Database tiers across distinct, non-overlapping Virtual Network subnets.
* **Controlled Public Ingress:** Expose only an Azure Standard Public Load Balancer to the public internet on TCP port 80.
* **Zero Public Exposure for Backend & Database:** The Application VM possesses no Public IP address; the Azure Database for MySQL Flexible Server operates with private access only inside a delegated subnet.
* **Controlled Outbound Internet Egress:** Provide package updates (`apt`), Node runtime packages (`npm`), and repository downloads via an Azure NAT Gateway without allowing unsolicited inbound traffic.
* **Bastion Jump Host Administration:** Restrict administrative SSH access to the Web VM from an authorized administrator IP (`/32`), and manage the private App VM exclusively via SSH ProxyJump (`-J`).
* **Resilient Internal Routing:** Route all frontend-to-backend API traffic through an Azure Standard Internal Load Balancer assigned a reserved private IP (`10.0.2.10:3001`).

---

## 2. End-to-End Architectural Blueprint

### Architecture Diagram
The diagram below illustrates the complete component topology, network boundaries, security perimeters, and bidirectional traffic flow.

![Azure Three-Tier Architecture Diagram](images/screenshot1_architecture_diagram.png)

### High-Level Topology Walkthrough
```mermaid
flowchart TD
    subgraph Internet ["Public Internet"]
        User["End-User Client Browser"]
        Admin["DevOps Workstation (Admin IP /32)"]
    end

    subgraph Azure ["Microsoft Azure Cloud (South Africa North) | BookReview_RG"]
        subgraph VNet ["Virtual Network: Book_Review_VNet (10.0.0.0/16)"]
            
            subgraph WebTier ["Presentation Tier: Web_Subnet (10.0.1.0/24)"]
                PubLB["Azure Public Load Balancer<br/>Frontend PIP :80"]
                WebVM["Web VM: book-review-web-vm (10.0.1.4)<br/>• Nginx Reverse Proxy (:80)<br/>• Next.js 15 SSR/SPA (:3000)"]
            end

            subgraph AppTier ["Application Tier: App_Subnet (10.0.2.0/25)"]
                IntLB["Azure Internal Load Balancer<br/>Private Frontend IP: 10.0.2.10 :3001"]
                AppVM["App VM: book-review-app-vm (10.0.2.4)<br/>• NO Public IP Address<br/>• Node.js / Express API (:3001)<br/>• Managed by PM2 Runtime"]
            end

            subgraph DBTier ["Data Tier: DB_Subnet (10.0.3.0/26)"]
                MySQL["Azure Database for MySQL Flexible Server<br/>• Private Access & DNS Zone<br/>• Port 3306 (SSL/TLS Required)<br/>• Database: book_review_db"]
            end

            NAT["Azure NAT Gateway: Book_Review_NAT_GW<br/>Public IP: Book_Review_PIP<br/>(Outbound Updates for Web & App Subnets)"]
        end
    end

    %% Ingress Flow
    User -->|HTTP :80| PubLB
    PubLB -->|Forward TCP :80| WebVM
    WebVM -->|Local Loopback Proxy :3000| WebVM
    WebVM -->|API Route /api/* :3001| IntLB
    IntLB -->|Forward TCP :3001| AppVM
    AppVM -->|MySQL Protocol :3306 (TLS Enforced)| MySQL

    %% Egress Flow
    WebVM -.->|Outbound Egress Only| NAT
    AppVM -.->|Outbound Egress Only| NAT

    %% Administrative Flow
    Admin -->|SSH :22 (Restricted to Admin /32)| WebVM
    Admin -.->|SSH ProxyJump -J via Web VM| AppVM
```

---

## 3. Network Segmentation & IP Addressing Scheme

The Virtual Network uses the classless inter-domain routing (CIDR) block `10.0.0.0/16`. It is segmented into three non-overlapping subnets sized according to workload characteristics, scalability requirements, and Azure-reserved IP constraints:

| Subnet Name | CIDR Block | Total IPs | Azure Reserved IPs | Usable Host IPs | Purpose & Associated Resources | Delegation |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **`Web_Subnet`** | `10.0.1.0/24` | 256 | 5 | 251 | Public Load Balancer backend pool, Web VM (`10.0.1.4`) | None |
| **`App_Subnet`** | `10.0.2.0/25` | 128 | 5 | 123 | Internal Load Balancer (`10.0.2.10`), Private App VM (`10.0.2.4`) | None |
| **`DB_Subnet`** | `10.0.3.0/26` | 64 | 5 | 59 | Azure Database for MySQL Flexible Server & Private Endpoints | `Microsoft.DBforMySQL/flexibleServers` |

> **Azure Subnet Address Reservation Rule:** In any Azure Virtual Network subnet, Azure reserves the first four IP addresses (`.0` network, `.1` default gateway, `.2`/`.3` Azure DNS mapping) and the last IP address (`.255` broadcast) for cloud platform operations.

---

## Task 1: Architecture Design & Implementation Strategy

### 1.1 Strategic Design Decisions
* **Cloud Region Selection:** `South Africa North`. Selected based on validated capacity for burstable compute (`Standard_B2s_v2` / `Standard_B2ats_v2`), Azure MySQL Flexible Server burstable SKUs (`Standard_B1ms`), and minimal latency.
* **Compute Capacity Planning:** Ubuntu 24.04 LTS instances running on dual-core burstable VMs (`Standard_B2s_v2`). This configuration provides adequate memory (4 GiB) and burstable CPU credits to compile Next.js production builds and execute Node.js Express workloads without memory starvation.
* **High Availability & Single-Instance Tradeoff:** For this deployment capstone, one Virtual Machine is deployed per compute tier. Azure Standard Load Balancers monitor target instances via TCP health probes. While a single VM per tier introduces a maintenance window during virtual machine restart, the architecture is designed to scale horizontally into Azure Virtual Machine Scale Sets (VMSS) across multiple Availability Zones without architectural refactoring.
* **Database Sizing & Data Integrity:** Azure Database for MySQL Flexible Server provisioned with 32 GiB storage and 1-day automated point-in-time recovery backup retention. Enforced SSL/TLS encryption (`require_secure_transport = ON`) guarantees protection against eavesdropping on internal database connections.
* **Secret Management Strategy:** Application secrets (database credentials, JWT signing secrets, database host FQDN) are maintained in local `.env` configuration files with file permissions hardened to `chmod 600` (readable and writable only by the application owner). Sensitive values are excluded from Git version control via `.gitignore`.
* **Disaster Recovery & Operational Continuity:** Azure MySQL automated backups support point-in-time restoration. Application runtimes are supervised by PM2 process manager configured with systemd startup services to ensure automatic recovery upon instance restart.

### 1.2 Evidence of Execution
> **Screenshot 1:** Full three-tier architecture diagram displaying the public entry point, tier boundaries, network security isolation, and data flow paths.

![Screenshot 1: Architecture Diagram](images/screenshot1_architecture_diagram.png)

---

## Task 2: Azure Network Foundation Deployment

### 2.1 Overview & Technical Purpose
Task 2 provisions the foundational networking infrastructure that supports all subsequent compute, load balancing, and database resources. Complete isolation between tiers is established at the network layer.

### 2.2 Provisioned Network Resources
The following resources were provisioned within the dedicated Resource Group:

1. **Dedicated Resource Group:**
   * **Name:** `BookReview_RG`
   * **Location:** `southafricanorth`
   * **Purpose:** Logical container encapsulating all network, compute, database, and load balancing assets for lifecycle and security management.

2. **Virtual Network (VNet):**
   * **Name:** `Book_Review_VNet`
   * **Address Space:** `10.0.0.0/16` (65,536 addresses)
   * **Location:** `southafricanorth`

3. **Subnet Segmentation & Delegation:**
   * **`Web_Subnet` (`10.0.1.0/24`):** Designated for the public-facing Nginx reverse proxy and Next.js frontend VM.
   * **`App_Subnet` (`10.0.2.0/25`):** Designated for the private Node.js/Express API VM and the Internal Load Balancer private frontend.
   * **`DB_Subnet` (`10.0.3.0/26`):** Dedicated exclusively to the Azure Database for MySQL Flexible Server. Delegated to `Microsoft.DBforMySQL/flexibleServers`, granting Azure Resource Provider permission to inject the managed database network interface directly into the private subnet.

4. **Internal Load Balancer Static IP Reservation:**
   * **Reserved IP:** `10.0.2.10` in `App_Subnet` (`10.0.2.0/25`).
   * **Purpose:** Serves as the static internal endpoint for API requests originating from Nginx on the Web VM. Excluded from dynamic VM allocation to prevent IP conflicts.

5. **Azure NAT Gateway & Public IP:**
   * **NAT Gateway Name:** `Book_Review_NAT_GW`
   * **Public IP Name:** `Book_Review_PIP` (Standard SKU, Static Allocation, `4.221.200.108`)
   * **Subnet Associations:** Associated with both `Web_Subnet` and `App_Subnet`.
   * **Architectural Rationale:** The App VM operates with **no Public IP address** to maintain strict isolation from external threats. To download operating system patches via `apt`, fetch application code from GitHub, and install dependencies via `npm`, it requires outbound Internet access. The Azure NAT Gateway translates outbound requests to `Book_Review_PIP` while strictly dropping any unsolicited inbound traffic from the Internet.

---

### 2.3 Verification & Evidence

#### Screenshot 2: Resource Group Overview
Shows the newly created `BookReview_RG` containing the core network foundation: `Book_Review_VNet`, `Book_Review_NAT_GW`, and `Book_Review_PIP`.

![Screenshot 2: Resource Group Overview](images/screenshot2_resource_group_overview.png)

#### Screenshot 3: Virtual Network Subnet Configuration
Displays `Book_Review_VNet` subnets showing CIDR ranges, usable addresses (`251` for Web, `123` for App, `59` for DB), and delegation of `DB_Subnet` to `Microsoft.DBforMySQL/flexibleServers`.

![Screenshot 3: VNet Subnets Configuration](images/screenshot3_vnet_subnets_delegation.png)

#### Screenshot 4: NAT Gateway Outbound Configuration & Subnet Association
Demonstrates the `Book_Review_NAT_GW` associated with the dedicated Public IP `Book_Review_PIP` (`4.221.200.108`) and attached to both `Web_Subnet` and `App_Subnet`.

![Screenshot 4: NAT Gateway Configuration](images/screenshot4_nat_gateway_configuration.png)

---

## Task 3: Security & Network Access Control

### 3.1 Network Security Group (NSG) Design Matrix
To enforce strict least-privilege network access between tiers, three dedicated Network Security Groups are created and associated directly to their respective subnets:

#### Web Tier NSG: `Book_Review_Web_NSG` (Associated to `Web_Subnet`)
| Priority | Rule Name | Port | Protocol | Source | Destination | Action | Purpose |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **100** | `Allow-HTTP-Inbound` | `80` | TCP | `Internet` (or `Any`) | `VirtualNetwork` | Allow | Public user browser access to Nginx |
| **110** | `Allow-SSH-Admin` | `22` | TCP | `Admin_IP/32` | `VirtualNetwork` | Allow | Bastion / Jump Host SSH management |
| **120** | `Allow-Azure-LB-Probe` | `80` | TCP | `AzureLoadBalancer` | `VirtualNetwork` | Allow | Health probe monitoring from Public LB |
| **Default** | `DenyAllInbound` | Any | Any | Any | Any | Deny | Azure default block rule |

#### Application Tier NSG: `Book_Review_App_NSG` (Associated to `App_Subnet`)
| Priority | Rule Name | Port | Protocol | Source | Destination | Action | Purpose |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **100** | `Allow-WebSubnet-to-App` | `3001` | TCP | `10.0.1.0/24` | `VirtualNetwork` | Allow | Internal API calls from Web VM / Internal LB |
| **110** | `Allow-SSH-from-WebVM` | `22` | TCP | `10.0.1.4/32` | `VirtualNetwork` | Allow | SSH ProxyJump administration from Web VM |
| **120** | `Allow-Azure-LB-Probe` | `3001` | TCP | `AzureLoadBalancer` | `VirtualNetwork` | Allow | Health probe monitoring from Internal LB |
| **Default** | `DenyAllInbound` | Any | Any | Any | Any | Deny | Azure default block rule (blocks Internet) |

#### Database Tier NSG: `Book_Review_DB_NSG` (Associated to `DB_Subnet`)
| Priority | Rule Name | Port | Protocol | Source | Destination | Action | Purpose |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **100** | `Allow-MySQL-from-AppSubnet` | `3306` | TCP | `10.0.2.0/25` | `VirtualNetwork` | Allow | MySQL queries from App VM backend |
| **Default** | `DenyAllInbound` | Any | Any | Any | Any | Deny | Blocks all non-App subnet connections |

---

### 3.2 Web Virtual Machine Provisioning (`book-review-web-vm`)

The Web Virtual Machine functions as both the public presentation host and the administrative bastion jump host.

#### Specifications:
* **VM Name:** `book-review-web-vm`
* **Resource Group:** `BookReview_RG`
* **Region:** `South Africa North`
* **Image:** Ubuntu Server 24.04 LTS (x64)
* **Compute Size:** `Standard_B2s_v2` (2 vCPUs, 4 GiB Memory)
* **Subnet:** `Web_Subnet` (`10.0.1.0/24`)
* **Private IP:** Static `10.0.1.4`
* **Public IP:** `pip-book-review-web-vm` (Standard SKU, Static)
* **Authentication:** SSH Public Key (`azureuser`)
* **NIC Security Group:** None (delegated to subnet NSG `Book_Review_Web_NSG`)

---

### 3.3 Task 3 Verification & Evidence

#### Screenshot 5: Network Security Group Rules Comparison
Shows the inbound rules configured across `Book_Review_Web_NSG`, `Book_Review_App_NSG`, and `Book_Review_DB_NSG` demonstrating least-privilege traffic flow.

*(Screenshot 5 placeholder - Inbound Security Rules across Web, App, and DB NSGs)*

#### Screenshot 6: Web Virtual Machine Overview & Network Interface
Displays `book-review-web-vm` running in `Web_Subnet` with its private IP (`10.0.1.4`) and public IP, with the subscription ID masked.

*(Screenshot 6 placeholder - Web VM Overview & Network Details)*

---

## Upcoming Implementation Phases (Tasks 4–8)

* **Task 4:** Deploy Azure Database for MySQL Flexible Server with Private DNS in `DB_Subnet`.
* **Task 5:** Deploy isolated Application VM (`book-review-app-vm`) in `App_Subnet` (no public IP), initialize schema, and configure PM2 backend service.
* **Task 6:** Configure Next.js frontend and Nginx reverse proxy on `book-review-web-vm`.
* **Task 7:** Configure Public Load Balancer (Port 80) and Internal Load Balancer (Port 3001), plus Azure Monitor diagnostic metrics and alerts.
* **Task 8:** End-to-end integration validation, database persistence verification, and controlled fault recovery test.

---

## Author & Engineering Profile

* **Engineer:** Jamiu Dahunsi  
* **Role:** Cloud & DevOps Engineer  
* **Specialization:** Enterprise Cloud Infrastructure, Network Security, Microservices & Three-Tier Deployments on Microsoft Azure

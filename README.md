# 🏗️ Highly Available and Scalable Three-Tier Architecture on AWS

## 📘 Project Description

This project implements a **highly available and scalable three-tier architecture** on AWS, purpose-built to host a **secure and performant web application**. The infrastructure is fully deployed within a **single VPC**, spanning **three Availability Zones (AZs)** to ensure fault tolerance, high availability, and scalability.

![ChatGPT Image Jun 25, 2025, 11_08_18 AM](https://github.com/user-attachments/assets/e93f67af-8134-4ab4-a99c-f6d00354bd4d)


The architecture includes the following **subnet segmentation**, distributed across all three AZs:

- **Subnet A:** Public Web Subnet (for ALBs)  
- **Subnet B:** Private Web Subnet (Frontend EC2 instances)  
- **Subnet C:** Private App Subnet (Backend EC2 instances, Internal ALB)  
- **Subnet D:** Private DB Subnet (RDS/Aurora)  

## 🔄 Architectural Flow

1. **User Access:**  
   Users access the application through a public ALB in Subnet A.

2. **Frontend Tier:**  
   Traffic is routed to EC2 instances (React/Node.js/NGINX) in Private Subnet B via Auto Scaling Group.

3. **Application Tier:**  
   Frontend communicates with an internal ALB in Subnet C, which forwards to backend EC2s (PHP/Logic).

4. **Database Tier:**  
   Backend servers interact with a managed database (RDS or Aurora) in Subnet D.

## 🧱 Infrastructure Components

### 1. 🕸️ Networking and Core Infrastructure
| **Service** | **Activity** | **Purpose** |
|-------------|--------------|-------------|
| **Amazon VPC** | Create a VPC | Provides isolated networking |
| **Subnets** | Create Public Web, Private Web, App, and DB subnets across AZs (1a, 1b, 1c) | Logical segmentation of infrastructure |
| **Route Tables** | Create and associate route tables per subnet group | Defines traffic rules per tier |
| **Internet Gateway (IGW)** | Attach to VPC and add route to public route table | Allows internet access for public subnets |
| **NAT Gateway** | Deploy in Public Subnet A and route for Private Subnets B/C | Allows private instances to access the internet securely |

### 2. 🔐 Security Configuration
| **Service** | **Activity** | **Purpose** |
|-------------|--------------|-------------|
| **Security Groups** | Create SGs for: Frontend ALB, Frontend EC2s, Backend ALB, Backend EC2s, DB | Controls tier-specific network access |

### 3. 🛢️ Database Tier
| **Service** | **Activity** | **Purpose** |
|-------------|--------------|-------------|
| **RDS** | Create DB Subnet Group and launch DB in Private Subnet D | Provides scalable, managed database instance in isolated network |

### 4. ⚖️ Load Balancing
| **Service** | **Activity** | **Purpose** |
|-------------|--------------|-------------|
| **Application Load Balancer (ALB)** | Create Frontend ALB (public) and Backend ALB (internal) | Distributes traffic to respective tier |
| **Target Groups** | Create for each ALB | Routes requests to appropriate EC2 ASG |

### 5. 💻 Compute Tier
| **Service** | **Activity** | **Purpose** |
|-------------|--------------|-------------|
| **EC2 AMIs** | Create custom AMIs: Frontend → NGINX, Git; Backend → PHP, Apache, MySQL client, Git, DB script | Preconfigured server templates |
| **Launch Templates** | Create for Frontend and Backend servers | Base config for ASG deployments |
| **Auto Scaling Group (ASG)** | Create ASGs for Frontend and Backend tiers | Ensures high availability and elasticity |

## 🧰 Technologies Used
| **Component** | **Technology/Service** |
|---------------|-------------------------|
| Compute        | Amazon EC2 with Auto Scaling |
| Load Balancing | Application Load Balancer (ALB) |
| Database       | Amazon RDS / Aurora (Multi-AZ) |
| Networking     | VPC, Subnets, NAT Gateway |
| Security       | Security Groups, NACLs, IAM |
| High Availability | Multi-AZ Deployment |
| Image Management | EC2 AMI + Launch Templates |

## 🚀 Deployment Strategy

This architecture can be provisioned using:
- AWS Management Console (manual)

# 🛠️ Implementation Guide

## PART 1: Networking and Core Infrastructure

### 🔹 Step 1: Create a VPC (Virtual Private Cloud)

1. **Navigate to the Amazon VPC Console**
   - Open the AWS Management Console
   - Go to **VPC** under **Networking & Content Delivery**

2. **Create a New VPC**
   - Click on **Create VPC**
   - Select **"VPC only"** option

3. **Configure the VPC Settings**
   - **Name tag:**  
     `demo-vpc` (or any custom name)
   - **IPv4 CIDR block:**  
     `10.0.0.0/16`
     - This range allows up to **65,536 IP addresses**
     - Suitable for small-to-medium scale infrastructures
   - **Leave all other settings as default**

4. **Click "Create VPC"**
![1](https://github.com/user-attachments/assets/108fe6bb-8b66-4762-80f9-1654f9df9c91)

### 🔹 Step 2: Create Subnets

You’ll be creating **12 subnets total**, across three Availability Zones:
- **3 Public Subnets** → for Load Balancers / NAT
- **3 Private Web Subnets** → for Frontend EC2 tier
- **3 Private App Subnets** → for Backend EC2 tier
- **3 Private DB Subnets** → for RDS / Aurora

> 🔁 **Note:** Each set of subnets is spread across **us-east-1a**, **us-east-1b**, and **us-east-1c** for high availability.

#### 🧩 Step 2.1: Create Public Web Subnets
1. Go to the **VPC Dashboard** → **Subnets**
2. Click **"Create subnet"**
3. Choose the VPC created in Step 1 (e.g., `demo-vpc`)
4. Add the following public subnets:

| Name                    | AZ           | CIDR Block     |
|-------------------------|--------------|----------------|
| web-public-subnet-1a    | us-east-1a   | 10.0.0.0/20    |
| web-public-subnet-1b    | us-east-1b   | 10.0.16.0/20   |
| web-public-subnet-1c    | us-east-1c   | 10.0.32.0/20   |

- Click **"Add new subnet"** for each
- Then click **"Create subnet"** when all are added

> 💡 Each `/20` block allows for ~4,096 IP addresses.

#### 🧩 Step 2.2: Create Private Web Subnets (Frontend Tier)
Repeat the process to create the private subnets for the frontend (web tier):

| Name                    | AZ           | CIDR Block     |
|-------------------------|--------------|----------------|
| web-private-subnet-1a   | us-east-1a   | 10.0.48.0/20   |
| web-private-subnet-1b   | us-east-1b   | 10.0.64.0/20   |
| web-private-subnet-1c   | us-east-1c   | 10.0.80.0/20   |

- Private subnets will **not** have a direct route to the Internet
- Click **"Create subnet"** when all three are defined

#### 🧩 Step 2.3: Create Private App Subnets (Backend Tier)
These subnets are for backend EC2 instances or microservices:

| Name                    | AZ           | CIDR Block     |
|-------------------------|--------------|----------------|
| app-private-subnet-1a   | us-east-1a   | 10.0.96.0/20   |
| app-private-subnet-1b   | us-east-1b   | 10.0.112.0/20  |
| app-private-subnet-1c   | us-east-1c   | 10.0.128.0/20  |

- Repeat the same process → Click **"Create subnet"** after entry

#### 🧩 Step 2.4: Create Private Database Subnets (DB Tier)
These subnets are reserved exclusively for **Amazon RDS** or **Aurora** instances. They are **isolated** from the public internet for maximum security.

1. Go to the **VPC Dashboard** → **Subnets**
2. Click **"Create subnet"**
3. Choose the VPC (`demo-vpc`)
4. Add the following database subnets:

| Name                   | Availability Zone | CIDR Block      |
|------------------------|-------------------|------------------|
| db-private-subnet-1a   | us-east-1a        | 10.0.144.0/20    |
| db-private-subnet-1b   | us-east-1b        | 10.0.160.0/20    |
| db-private-subnet-1c   | us-east-1c        | 10.0.176.0/20    |

- Click **"Create subnet"** when all three entries are complete
> 🔒 These subnets **do not** need a NAT Gateway or IGW access — they remain isolated unless specific access is granted via Security Groups.

### ✅ Subnet Summary Table

Here's a full overview of the 12 subnets created across three Availability Zones:

| **Subnet Type**         | **AZ 1 (1a)**     | **AZ 2 (1b)**     | **AZ 3 (1c)**     |
|--------------------------|------------------|------------------|------------------|
| Public Web Subnet        | 10.0.0.0/20       | 10.0.16.0/20      | 10.0.32.0/20      |
| Private Web Subnet       | 10.0.48.0/20      | 10.0.64.0/20      | 10.0.80.0/20      |
| App Private Subnet       | 10.0.96.0/20      | 10.0.112.0/20     | 10.0.128.0/20     |
| DB Private Subnet        | 10.0.144.0/20     | 10.0.160.0/20     | 10.0.176.0/20     |

> 🌐 **Total Subnets Created:** 12
![2](https://github.com/user-attachments/assets/7b61ec59-2bc2-41b1-a46d-16d5e1720460)

> ✅ All CIDR blocks are non-overlapping and evenly distributed across Availability Zones

---

### 🔹 Step 3: Configure Route Tables and Internet Access

Each of the 12 subnets will have its **own dedicated route table**, enabling:

- 🔄 **Independent routing per AZ** (ideal for multi-NAT, multi-AZ failover)
- 🔒 **Security segregation** per tier (Web, App, DB)
- 🌐 **Controlled internet access** (only public/web tiers route to the internet)

> ⚙️ **Best Practice:** One route table per subnet gives you the flexibility to scale, secure, and isolate routing behavior efficiently.

### 🌐 Why One Shared Route Table for Public Subnets?

- All three public subnets require **identical routing** to an **Internet Gateway (IGW)**
- A single **public route table** is created and associated with all three public subnets
- This route table will include a route:  
  `0.0.0.0/0 → Internet Gateway`

### 🛣️ Step 3.1: Create Public Route Table

1. Go to **VPC Dashboard** → **Route Tables**
2. Click **Create route table**
3. Configure:
   - **Name:** `web-public-route-table`
   - **VPC:** Select `demo-vpc`
4. Click **Create route table**
5. Edit routes:
   - Add route:  
     `Destination: 0.0.0.0/0`  
     `Target: <Internet Gateway ID>`
6. Associate this route table with:
   - `web-public-subnet-1a`
   - `web-public-subnet-1b`
   - `web-public-subnet-1c`

### 🔒 Step 3.2: Create Private Route Tables for Web Tier

Create 1 route table per AZ for **frontend private web-tier instances**.

| Name                      | AZ         |
|---------------------------|------------|
| web-private-route-table-1a | us-east-1a |
| web-private-route-table-1b | us-east-1b |
| web-private-route-table-1c | us-east-1c |

> These route tables will later point to **NAT Gateways** for secure outbound access.

### ⚙️ Step 3.3: Create Private Route Tables for App Tier

Used by the **application EC2 instances** in Private App Subnets:

| Name                      | AZ         |
|---------------------------|------------|
| app-private-route-table-1a | us-east-1a |
| app-private-route-table-1b | us-east-1b |
| app-private-route-table-1c | us-east-1c |

> These also route outbound via NAT for updates/API access.

### 🧱 Step 3.4: Create Private Route Tables for Database Tier
Strictly internal, **no internet access** (outbound or inbound):

| Name                      | AZ         |
|---------------------------|------------|
| db-private-route-table-1a | us-east-1a |
| db-private-route-table-1b | us-east-1b |
| db-private-route-table-1c | us-east-1c |

> 🚫 Do **not** add internet or NAT routes here. These are used for **RDS**, which should be isolated.

![3](https://github.com/user-attachments/assets/bbe3f0a8-162b-428a-8430-6ed59a06faee)

### ✅ Route Table Summary

| Route Table Name            | Associated Tier      | Availability Zone | Internet Access |
|-----------------------------|----------------------|-------------------|-----------------|
| web-public-route-table      | Public Web           | All 3 AZs         | ✅ Yes (via IGW) |
| web-private-route-table-1a  | Web Tier (Private)   | us-east-1a        | ✅ (via NAT)     |
| web-private-route-table-1b  | Web Tier (Private)   | us-east-1b        | ✅ (via NAT)     |
| web-private-route-table-1c  | Web Tier (Private)   | us-east-1c        | ✅ (via NAT)     |
| app-private-route-table-1a  | App Tier             | us-east-1a        | ✅ (via NAT)     |
| app-private-route-table-1b  | App Tier             | us-east-1b        | ✅ (via NAT)     |
| app-private-route-table-1c  | App Tier             | us-east-1c        | ✅ (via NAT)     |
| db-private-route-table-1a   | Database Tier        | us-east-1a        | ❌ No            |
| db-private-route-table-1b   | Database Tier        | us-east-1b        | ❌ No            |
| db-private-route-table-1c   | Database Tier        | us-east-1c        | ❌ No            |

---

### 🔹 Step 4: Associate Route Tables with Subnets

---

#### 🔍 Why Associate Route Tables with Subnets?

Every subnet must be explicitly associated with a route table to define how traffic is routed. Doing this allows you to:

- ✅ Apply **custom routing behavior** per subnet (e.g., public vs. private)
- ✅ Support **NAT Gateway per AZ** (for zone-specific failover)
- ✅ Maintain **tier-based isolation and control**
- ✅ Improve **scalability and security posture**

---

### 🔧 Step 4.1: Associate `web-public-route-table` with All Public Subnets

1. Navigate to **VPC Dashboard** → **Route Tables**
2. Select **`web-public-route-table`**
3. In the **bottom panel**, choose the **“Subnet Associations”** tab
4. Click **“Edit subnet associations”**
![4](https://github.com/user-attachments/assets/9db38b46-9e40-4a03-870d-611c8ddb4297)
5. Check the boxes for:
   - `web-public-subnet-1a`
   - `web-public-subnet-1b`
   - `web-public-subnet-1c`
6. Click **“Save associations”**
![5](https://github.com/user-attachments/assets/23ae0e0c-bffc-46f9-b858-46abb993a277)
### ✅ What Happens After Associating Route Tables?
- The associated subnets will **inherit the routing rules** defined in their route table.
- For example, the **public subnets** associated with `web-public-route-table` will:
  - Follow the route `0.0.0.0/0 → Internet Gateway (IGW)`
  - Gain **direct internet access**, making them publicly accessible.
- This setup is typically used for resources that require internet exposure, such as:
  - **Application Load Balancers (ALBs)**
  - **Bastion Hosts**
  - Other internet-facing services.

> 🔁 Repeat the following process for each tier, matching route tables to subnets in the **same AZ**.

### 🛠️ Step 4.2: Associate Web Private Route Tables
| Route Table Name            | Subnet Name             |
|-----------------------------|--------------------------|
| `web-private-route-table-1a` | `web-private-subnet-1a` |
| `web-private-route-table-1b` | `web-private-subnet-1b` |
| `web-private-route-table-1c` | `web-private-subnet-1c` |

### 🛠️ Step 4.3: Associate App Private Route Tables
| Route Table Name            | Subnet Name             |
|-----------------------------|--------------------------|
| `app-private-route-table-1a` | `app-private-subnet-1a` |
| `app-private-route-table-1b` | `app-private-subnet-1b` |
| `app-private-route-table-1c` | `app-private-subnet-1c` |

### 🛠️ Step 4.4: Associate DB Private Route Tables
| Route Table Name            | Subnet Name             |
|-----------------------------|--------------------------|
| `db-private-route-table-1a` | `db-private-subnet-1a`  |
| `db-private-route-table-1b` | `db-private-subnet-1b`  |
| `db-private-route-table-1c` | `db-private-subnet-1c`  |

> 📌 Every subnet is now associated with its intended route table, giving a granular control and resilience across AZs.
> ⚠️ Note: Internet access is only possible **after** attaching an IGW and configuring the correct routes, which is covered in the next steps.

---

### 🔹 Step 5: Create Internet Gateway (IGW) and NAT Gateway

#### 🔍 What are IGW and NAT Gateway?
- **Internet Gateway (IGW):**  
  A horizontally scaled, redundant, and highly available gateway that enables communication between the VPC and the internet. It allows public subnets to send and receive traffic to/from the internet.

- **NAT Gateway:**  
  Allows instances in **private subnets** to initiate outbound internet traffic (for updates, API calls, etc.) while **blocking inbound traffic** from the internet. This keeps private subnets secure while allowing necessary outbound access.

#### 🔧 Step 5.1: Create Internet Gateway
1. Open the **VPC Dashboard** → **Internet Gateways**
2. Click **Create Internet Gateway**
3. Enter a name tag, e.g., `demo-internet-gateway`
4. Click **Create**
![6](https://github.com/user-attachments/assets/ca6d8b5a-204a-4cd7-8f48-2bb1bbd0a496)

#### 🔧 Step 5.2: Attach Internet Gateway to the VPC
1. Select the newly created `demo-internet-gateway`
2. From the **Actions** dropdown, choose **Attach to VPC**
3. Select the VPC (`demo-vpc` from Step 1)
4. Click **Attach Internet Gateway**
![7](https://github.com/user-attachments/assets/069ed0f0-abbf-4169-b4ed-16b1161ece8e)

> 🔑 **Why attach it?**  
> Attaching the IGW allows the **public subnets** to route traffic to and from the internet. Without this, the IGW exists but is ineffective since it’s not connected to the VPC.

#### 🔧 Step 5.3: Create NAT Gateway
1. Go to **VPC Dashboard** → **NAT Gateways**
2. Click **Create NAT Gateway**
3. Name it, e.g., `demo-nat-gateway`
4. Select a **Public Subnet** (e.g., `web-public-subnet-1a`)  
   > NAT Gateways must reside in a public subnet to access the internet.
5. For **Connectivity Type**, choose **Public**
6. Allocate or select an **Elastic IP address**  
   > An Elastic IP is a static public IPv4 address allowing the NAT Gateway to communicate with the internet.
7. Click **Create NAT Gateway**
![8](https://github.com/user-attachments/assets/6f6444ff-77e5-443b-8e43-df73dbc34064)

#### 🔄 Additional Notes on NAT Gateway
- In this project, we create a **single NAT Gateway** in one public subnet.
- For **production environments**, best practice is to deploy a NAT Gateway **in each Availability Zone** (i.e., one per public subnet across 1a, 1b, 1c).
- This multi-AZ setup ensures **fault tolerance and high availability**; if one NAT Gateway or AZ fails, others continue handling outbound traffic from private subnets.

---

### 🔹 Step 6: Add Routes to Internet Gateway and NAT Gateway
Just creating an Internet Gateway or NAT Gateway doesn’t automatically enable internet access. You must explicitly configure routing rules in the route tables to direct traffic appropriately.

#### Step 6.1: Add IGW Route to the Public Route Table
1. Open **VPC Console** → **Route Tables**
2. Select the public route table, e.g., `web-public-route-table`
3. Go to the **Routes** tab
![9](https://github.com/user-attachments/assets/bb6814cb-53ab-4729-9ce7-71d224b4de80)

4. Click **Edit Routes** → **Add Route**
5. Configure:
   - **Destination:** `0.0.0.0/0` (all IPv4 traffic)
   - **Target:** Select **Internet Gateway**, then choose the IGW (`demo-internet-gateway`)
6. Click **Save changes**
![10](https://github.com/user-attachments/assets/60490dc5-5273-46b4-b2bd-77ab463dde47)

> ✅ **Why?**  
> This route directs all external traffic from instances in public subnets to flow through the Internet Gateway, making the subnet publicly accessible.

#### Step 6.2: Add NAT Gateway Route to All Private Route Tables
Repeat the following for **all 9 private route tables** (Web, App, DB tiers):
1. Go to **Route Tables** in the VPC dashboard
2. Select a private route table (e.g., `web-private-route-table-1a`)
3. Click **Routes** → **Edit Routes** → **Add Route**
![11](https://github.com/user-attachments/assets/44effe40-94ff-491e-a013-66827b946714)

4. Configure:
   - **Destination:** `0.0.0.0/0` (all IPv4 traffic)
   - **Target:** Select **NAT Gateway**, then choose the NAT Gateway created in Step 5.3 (`demo-nat-gateway`)
5. Click **Save changes**

| Route Table Name           | Associated Subnet          |
|---------------------------|---------------------------|
| web-private-route-table-1a | web-private-subnet-1a      |
| web-private-route-table-1b | web-private-subnet-1b      |
| web-private-route-table-1c | web-private-subnet-1c      |
| app-private-route-table-1a | app-private-subnet-1a      |
| app-private-route-table-1b | app-private-subnet-1b      |
| app-private-route-table-1c | app-private-subnet-1c      |
| db-private-route-table-1a  | db-private-subnet-1a       |
| db-private-route-table-1b  | db-private-subnet-1b       |
| db-private-route-table-1c  | db-private-subnet-1c       |

> ✅ **Why use NAT Gateway for private subnets?**  
> Private instances need outbound internet access (e.g., for software updates, external API calls) but must **never be directly reachable from the internet**.  
> NAT Gateway enables secure, one-way internet access.

### 🛡️ Important Best Practice Reminder:
- In production, create **one NAT Gateway per Availability Zone**.
- Configure each private subnet to use the NAT Gateway in its own AZ for **high availability and fault tolerance**.

---

## Part 2: Security Configuration
To improve security and isolate tiers, we define **five security groups**, one for each component:

| Component            | Security Group Name    | Purpose / Description                          |
|----------------------|-----------------------|-----------------------------------------------|
| Frontend ALB         | `frontend-alb-sg`      | Allows inbound HTTP (port 80) from anywhere (public internet) |
| Frontend Servers      | `frontend-server-sg`   | Allows HTTP traffic from `frontend-alb-sg` only (not public)   |
| Backend ALB           | `backend-alb-sg`       | Allows traffic from `frontend-server-sg` only                 |
| Backend Servers       | `backend-server-sg`    | Allows traffic from `backend-alb-sg` only                     |
| Database              | `database-sg`          | Allows MySQL (port 3306) traffic from `backend-server-sg` only |

### Step 1.1: Frontend ALB Security Group
- **Name:** `frontend-alb-sg`
- **Description:** Security group for front-end ALB to allow inbound HTTP traffic from public internet.
- **VPC:** `demo-vpc`
- **Inbound Rule:**
  - Type: Custom TCP
  - Port Range: 80
  - Source: `0.0.0.0/0`
![13](https://github.com/user-attachments/assets/74b13f22-1f0b-440f-8d24-e48d5f0c3d28)

> ❓ Why?  
> Port 80 is standard HTTP. Allowing from anywhere enables public access to the ALB.

### Step 1.2: Frontend Servers Security Group
- **Name:** `frontend-server-sg`
- **Description:** Allows inbound HTTP traffic from `frontend-alb-sg`.
- **VPC:** `demo-vpc`
- **Inbound Rule:**
  - Type: Custom TCP
  - Port Range: 80
  - Source: `frontend-alb-sg`
![14](https://github.com/user-attachments/assets/8810ca58-4043-4c36-a52b-b150e58100b1)

> ❓ Why?  
> Only ALB can send traffic to frontend servers, improving security by blocking direct internet access.

### Step 1.3: Backend ALB Security Group
- **Name:** `backend-alb-sg`
- **Description:** Allows inbound HTTP traffic from `frontend-server-sg`.
- **VPC:** `demo-vpc`
- **Inbound Rule:**
  - Type: Custom TCP
  - Port Range: 80
  - Source: `frontend-server-sg`
![15](https://github.com/user-attachments/assets/015531dc-e676-4295-9471-55f09502c8be)

> ❓ Why?  
> Ensures backend ALB only receives traffic from trusted frontend servers, maintaining tier isolation.

### Step 1.4: Backend Servers Security Group
- **Name:** `backend-server-sg`
- **Description:** Allows inbound HTTP traffic from `backend-alb-sg`.
- **VPC:** `demo-vpc`
- **Inbound Rule:**
  - Type: Custom TCP
  - Port Range: 80
  - Source: `backend-alb-sg`
![16](https://github.com/user-attachments/assets/60ef5d26-32fc-4e87-972c-3f6a7f060ffc)

> ❓ Why?  
> Limits backend server access to traffic routed through backend ALB only. No direct internet or frontend access.

### Step 1.5: Database Security Group
- **Name:** `database-sg`
- **Description:** Allows MySQL traffic from `backend-server-sg` only.
- **VPC:** `demo-vpc`
- **Inbound Rule:**
  - Type: Custom TCP
  - Port Range: 3306
  - Source: `backend-server-sg`
![17](https://github.com/user-attachments/assets/45d89418-fbf5-4b00-bf44-fb9510593d08)

> ❓ Why?  
> Backend servers only can access the database; frontends and ALBs cannot. Limits attack surface.

## Step 3.1: Create RDS Subnet Group
An **RDS Subnet Group** ensures RDS instances are deployed in private subnets across multiple AZs, enabling multi-AZ support and isolation from the internet.

### How to Create:
1. Go to **RDS Dashboard** → **Subnet Groups** → **Create DB Subnet Group**
2. Configure:
   - **Name:** `db-subnet-group`
   - **Description:** `db-subnet-group`
   - **VPC:** Select `demo-vpc`
3. Add Subnets (one per AZ):
   - `us-east-1a`: select `db-private-subnet-1a`
   - `us-east-1b`: select `db-private-subnet-1b`
   - `us-east-1c`: select `db-private-subnet-1c`
4. Click **Create DB Subnet Group**
![18](https://github.com/user-attachments/assets/6319965b-2d77-439d-8dae-68021a1dbc0e)

> ✅ After this:  
> the RDS instances will deploy privately in these subnets, isolated from the public internet and ready for multi-AZ deployments for high availability.

---

## Part 3: Database Tier – Launch RDS Instance (MySQL)
This step creates a highly available MySQL RDS instance using the previously created subnet group and security group. Key goals include:
- Multi-AZ deployment for high availability and automatic failover.
- Deploy in private subnets only to prevent public exposure.
- Use a cost-effective burstable instance type (`db.t3.micro`) suitable for dev/test environments.

### Step-by-Step Guide to Create RDS Instance
1. **Navigate to:**  
   AWS Console → RDS → Databases → Create database

2. **Choose Creation Method:**  
   - Select **Standard Create** for full control.

3. **Engine Options:**  
   - Engine: **MySQL**  
   - Version: Leave default unless a specific version is needed.
![19](https://github.com/user-attachments/assets/87e5a7df-6e25-4634-8532-36692d7dbd6c)

4. **Templates:**  
   - Select **Dev/Test** for a simplified, cost-optimized setup.

5. **Availability & Durability:**  
   - Enable **Multi-AZ DB instance deployment** (deploys across 2 AZs).  
   > ✅ Ensures automatic failover and replication, critical for production readiness.

6. **Settings:**  
   - DB Instance Identifier: `demo-database`  
   - Master Username: `admin`  
   - Password: Enter a strong password manually under **Self-managed credentials**.
![20](https://github.com/user-attachments/assets/953ad508-e188-4d65-b555-5c12d6127c49)

7. **DB Instance Class:**  
   - Choose **Burstable classes** → Instance type: `db.t3.micro`  
   > 💸 Cost-effective for lightweight dev/test usage.
![21](https://github.com/user-attachments/assets/efaddba8-64dc-42a6-897c-3277ce3ca89a)

8. **Storage:**  
   - Allocated storage: **20 GiB** (minimum recommended, can be autoscaled optionally).

9. **Network & Security:**  
   - VPC: Select `demo-vpc` (created in Part 1)  
   - DB Subnet Group: Select `db-subnet-group` (created earlier)  
   - Public Access: **No** (default, for private deployment)  
   - VPC Security Group: Choose `database-sg` (created in Part 2)  
   > 🔐 Ensures only backend EC2 instances with proper security group access can connect.
![22](https://github.com/user-attachments/assets/8788b9de-6a2a-46b2-b1ff-65e9a3b59605)

10. **Additional Settings:**  
    - Leave defaults unless backups or logs are required for the use case.  
    - Optionally disable backups to save cost in non-production environments.

11. **Create Database:**  
    - Click **Create Database** to start provisioning.

### After Deployment
- The RDS instance will be deployed across two Availability Zones for fault tolerance.
- It resides only in private subnets, inaccessible from the internet.
- Access to the database is restricted to backend EC2 instances via security groups (`backend-server-sg` → `database-sg` on port 3306).
- Use the RDS-generated endpoint in the backend application or MySQL clients to connect securely.

---

## Part 5: Load Balancing
Load balancing improves scalability, availability, and security by distributing incoming traffic across multiple instances.

### Step 5.1: Frontend ALB (Application Load Balancer)
- **Purpose:** Handles public internet traffic, distributing requests across frontend servers in public subnets.
- **Configuration Steps:**

1. Go to **EC2 Dashboard → Load Balancing → Load Balancers → Create Load Balancer → Application Load Balancer**.
2. Set:
   - Load Balancer Name: `frontend-alb` (unique)
   - Scheme: **internet-facing** (public)
   - VPC: `demo-vpc`
![23](https://github.com/user-attachments/assets/3d3f6af0-5719-4b92-98e9-8568d6bb61bb)

   - Availability Zones: Select all 3 AZs with public subnets:
     - `us-east-1a → web-public-subnet-1a`
     - `us-east-1b → web-public-subnet-1b`
     - `us-east-1c → web-public-subnet-1c`
   - Security Group: `frontend-alb-sg` (allows inbound HTTP from anywhere)
![24](https://github.com/user-attachments/assets/9e4b01e4-66f5-41a6-a1d0-2990c8d1c5db)

3. Listener & Routing:
   - Listener: HTTP on port 80
   - Default Action: Create a new target group:
     - Target type: **Instances**
     - Target Group Name: `frontend-alb-tg`
     - Leave other settings as default
![25](https://github.com/user-attachments/assets/aee61d81-8482-49b6-9bd6-2eb2ff36c143)

4. Finalize:
   - Set default action to target group `frontend-alb-tg`
   - Review and **Create Load Balancer**
   - ![26](https://github.com/user-attachments/assets/d25d798f-02f6-40c7-a47f-bdc2a422579a)

> Note: Instances are registered dynamically later via Auto Scaling Groups.

### Step 5.2: Backend ALB (Internal Application Load Balancer)
- **Purpose:** Handles internal traffic between frontend and backend tiers, distributing requests to backend servers in private subnets, improving security by isolating backend services.

- **Configuration Steps:**
1. Go to **EC2 Dashboard → Load Balancing → Load Balancers → Create Load Balancer → Application Load Balancer**.
2. Set:
   - Load Balancer Name: `backend-alb` (unique)
   - Scheme: **internal** (private)
   - VPC: `demo-vpc`
   - Availability Zones: Select all 3 AZs with private app subnets:
     - `us-east-1a → app-private-subnet-1a`
     - `us-east-1b → app-private-subnet-1b`
     - `us-east-1c → app-private-subnet-1c`
   - Security Group: `backend-alb-sg` (controls allowed inbound traffic)
![27](https://github.com/user-attachments/assets/bff054c6-e998-4d43-81c1-bb576f91a3ed)

3. Listener & Routing:
   - Listener: HTTP on port 80
   - Default Action: Create a new target group:
     - Target type: **Instances**
     - Target Group Name: `backend-alb-tg`
     - Leave other settings default
4. Finalize:
   - Set default action to target group `backend-alb-tg`
   - Review and **Create Load Balancer**
![28](https://github.com/user-attachments/assets/50d4788c-8177-4384-9682-fe4bcd7b6f0c)

### Summary & Key Points
- **Frontend ALB:**  
  Internet-facing, accepts client traffic from the public internet, forwards requests to frontend servers.

- **Backend ALB:**  
  Internal-only, routes traffic from frontend servers to backend servers within private subnets, keeping backend services inaccessible from outside the VPC.

- **Benefits:**  
  - Fine-grained security and access control across tiers.  
  - Scalability with dynamic routing via target groups.  
  - Improved fault tolerance and traffic management, especially when combined with Auto Scaling Groups.

---

## Part 5: Compute Tier — AMI Creation
### Overview
Create custom AMIs for frontend and backend servers by launching temporary EC2 instances, installing required software, and then baking the AMIs. These AMIs will be used later to create Auto Scaling Groups and Launch Templates.

### Step 1: Launch Frontend Server Instance for AMI
- **Purpose:** Launch an instance to install NGINX, Git, and prepare frontend server image.

- **Configuration:**
| Parameter           | Value                      |
|---------------------|----------------------------|
| Instance Name       | frontend-server-ami         |
| AMI                 | Amazon Linux 2023           |
| Key Pair            | Create or use existing SSH key pair |
| VPC                 | demo-vpc                   |
| Subnet              | Public subnet (e.g., web-public-subnet-1a) |
| Auto-assign Public IP| Enabled                    |
| Security Group      | frontend-server-sg          |

- **Why?** Public subnet + public IP needed for software installation and updates. Temporary instance with SSH access for manual setup.

### Step 2: Launch Backend Server Instance for AMI
- **Purpose:** Prepare backend server image including Apache, PHP, MySQL client, Git, and DB init scripts.

- **Configuration:**
| Parameter           | Value                      |
|---------------------|----------------------------|
| Instance Name       | backend-server-ami          |
| AMI                 | Amazon Linux 2023           |
| Key Pair            | Same as frontend instance   |
| VPC                 | demo-vpc                   |
| Subnet              | Public subnet (e.g., web-public-subnet-1a) |
| Auto-assign Public IP| Enabled                    |
| Security Group      | backend-server-sg           |

- **Why?** Same reason as frontend: temporary public access for package installation and script setup.

### Step 3: Temporarily Enable SSH Access for Setup
- **Step 3.1: Update `frontend-server-sg`**
  - Add inbound rule:
    - Type: SSH
    - Protocol: TCP
    - Port Range: 22
    - Source: 0.0.0.0/0 (temporary for setup)
![29](https://github.com/user-attachments/assets/64296dce-c7a8-4cc6-a153-555677ba7ea2)

- **Step 3.2: Update `backend-server-sg`**
  - Add inbound rule:
    - Type: SSH
    - Protocol: TCP
    - Port Range: 22
    - Source: 0.0.0.0/0 (temporary)

- **Important:**  
  Remove these SSH rules from both security groups immediately after AMI creation for security best practices.

## ✅ Step 4: Connect to Instances via EC2 Instance Connect
The goal here is to access the frontend and backend EC2 instances using the browser-based EC2 Instance Connect, so you can install required packages.

### 🖥️ 4.1: Connect to Frontend Instance
- Navigate to **EC2 Dashboard → Instances**
- Select the instance named `frontend-server-ami`
- Click **Connect**
- Choose the **EC2 Instance Connect (browser-based SSH)** tab
- Click **Connect**
![30](https://github.com/user-attachments/assets/1b3bb4e9-1bd0-43c6-8f3e-4883cef47ad6)

### 🖥️ 4.2: Connect to Backend Instance
- Navigate to **EC2 Dashboard → Instances**
- Select the instance named `backend-server-ami`
- Click **Connect**
- Choose the **EC2 Instance Connect (browser-based SSH)** tab
- Click **Connect**

## Step 5: Install & Configure Software on Frontend Server
### 5.1 Install Required Packages
```bash
# Update system and install NGINX
sudo dnf install nginx -y

# Install Git
sudo dnf install git -y
```
**Nginx**: Serves frontend static content and proxies API requests.
**Git:** For pulling app code or assets.

### 5.2 Configure NGINX
1. Open the NGINX config file:
   ```bash
   sudo vi /etc/nginx/nginx.conf
2. Press `i` to enter insert mode.
3. Comment out the existing `server { ... }` block by adding `#` at the start of each line.
4. Paste the following configuration **after** the commented section:
![Screenshot from 2025-06-25 01-38-02](https://github.com/user-attachments/assets/10e3cfb6-d8c8-471b-a7db-6239d395fc41)

server {
    listen 80;
    server_name localhost;
    root /usr/share/nginx/html;
    index index.html;

    # Static files caching
    location ~* \.(jpg|jpeg|png|gif|ico|css|js)$ {
        expires 30d;
        add_header Cache-Control "public, no-transform";
    }

    # API requests forwarded to backend
    location /api/ {
        proxy_pass http://update-me;  # Replace with backend ALB DNS/IP
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # SPA fallback
    location / {
        try_files $uri $uri/ /index.html;
    }
}
**Important:** Replace `http://update-me` with the backend ALB DNS name or internal IP once available.
**Save and exit:**
1. Press `Esc`  
2. Type `:wq`  
3. Hit `Enter`

## Step 6.1: Install & Configure LAMP Stack on Backend Server
Connect to the `backend-server-ami` instance using EC2 Instance Connect.
📦 Run the following commands step-by-step:
1. **Update all system packages**

```bash
sudo dnf upgrade -y
```
This ensures the instance has the latest security and bug fixes.

2. **Install Apache web server and essential PHP packages**
```bash
sudo dnf install -y httpd wget php-fpm php-mysqli php-json php php-devel
```
Installs:
- **Apache (httpd)** – the web server  
- **PHP & modules** – required for processing dynamic pages and connecting to MySQL

3. Install the MySQL client (MariaDB 10.5)
```bash
sudo dnf install mariadb105-server
```
Adds MySQL tools so the backend can interact with a remote database server (e.g., RDS).

4. Start the Apache service
```bash
sudo systemctl start httpd
```
Starts the web server immediately.

5. Enable Apache to start on reboot
```bash
sudo systemctl enable httpd
```
Ensures the web server stays active across restarts.

6. Add ec2-user to the Apache group
```bash
sudo usermod -a -G apache ec2-user
```
Grants the default EC2 user the ability to edit and manage Apache files.

7. Change ownership of the web root directory
```bash
sudo chown -R ec2-user:apache /var/www
```
Gives ec2-user ownership of the web content directory.

8. Set correct permissions for the /var/www directory and subdirectories
```bash
sudo chmod 2775 /var/www && find /var/www -type d -exec sudo chmod 2775 {} \;
```
Ensures the directory is writable by the owner and group, and that new files/folders inherit permissions.

9. Install Git
```bash
sudo dnf install git -y
```
Allows you to pull the backend application or deployment scripts from a Git repository.
✅ **Done!**

the backend EC2 instance is now ready to:
- Run Apache and PHP  
- Connect to a database  
- Pull code from Git  
- Be baked into an AMI for future use

## Step 6.2: Connect Backend to RDS & Initialize Database
### 1. Retrieve RDS Endpoint
- Go to **RDS Dashboard → Databases**
- Select the RDS instance
- Copy the **Endpoint** (e.g., `database-1.cjac84smcziu.us-east-1.rds.amazonaws.com`)
- Note the port number (default: `3306`)
![Screenshot from 2025-06-25 02-00-51](https://github.com/user-attachments/assets/64d10e3c-b128-4b04-b198-bf3bd390d0f5)

### 2. Connect to RDS from backend EC2 instance
```bash
mysql -h <RDS-endpoint> -u admin -p
```
Replace `<RDS-endpoint>` with the actual database endpoint.
Enter the password when prompted.
![Screenshot from 2025-06-25 02-05-07](https://github.com/user-attachments/assets/1dff3f33-9b89-4dfb-adb2-c67b4bd86037)
You should now be logged into the MySQL database shell.

📜 **3. Run SQL Script to Initialize the Database**
Paste the following SQL lines to run:
```sql
-- Create database
CREATE DATABASE IF NOT EXISTS hello_world;

-- Use the new database
USE hello_world;

-- Create messages table
CREATE TABLE IF NOT EXISTS messages (
    id INT AUTO_INCREMENT PRIMARY KEY,
    message VARCHAR(255) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Insert initial records
INSERT INTO messages (message) VALUES
('Hello from the database!'),
('Welcome to our three-tier architecture demo'),
('This is a simple example showing frontend, backend, and database');
```
![Screenshot from 2025-06-25 02-08-23](https://github.com/user-attachments/assets/17a6e110-6f95-4dd9-9e14-97575ba9751c)

 **Explanation of the SQL Script**
- Creates a database named `hello_world` if it doesn't already exist.
- Defines a `messages` table to store short messages with timestamps.
- Inserts three demo rows to verify setup and connectivity.

🚪 **4. Exit MySQL**
```bash
exit
```
💡 **5. Install Git (if not already done)**
```bash
sudo dnf install git -y
```
✅ Done! the backend server can now:
- Connect to the RDS MySQL database
- Use a properly initialized schema
- Pull application code via Git (if needed)

# Step 7: Create AMIs for Frontend & Backend Servers

### 💡 Why Stop the Instances First?
Stopping the EC2 instances before creating AMIs ensures:
- File system consistency (especially for dynamic services like MySQL or Apache logs)
- Minimal data corruption risk
- A clean snapshot of the instance without active writes
> 🔹 It’s not required, but highly recommended for production-quality AMIs.

### 🛑 Stop Both Instances
From the EC2 Dashboard:
1. Select both `frontend-server-ami` and `backend-server-ami`
2. Click **Instance State → Stop instance**
3. Wait for the status to change to **stopped**
![Screenshot from 2025-06-25 02-10-49](https://github.com/user-attachments/assets/75c69cb9-c044-48c0-8ff6-f5f388d3676e)

![Screenshot from 2025-06-25 02-13-27](https://github.com/user-attachments/assets/ba0a55bc-1df9-43da-ab9e-4c709bb75f32)

### 📸 Step 7.1: Create AMI for Frontend Server
1. Go to **EC2 Dashboard → Instances**
2. Select `frontend-server-ami`
![Screenshot from 2025-06-25 02-14-43](https://github.com/user-attachments/assets/68c93cb5-4643-423b-a66c-e6e32f085b1b)

3. Click **Actions → Image and templates → Create image**
4. Name: `frontend-server-ami`
5. (Optional) Add description or tags
6. Click **Create image**
> ✅ AWS will now create an AMI from this stopped instance.

### 📸 Step 7.2: Create AMI for Backend Server
1. Select `backend-server-ami`
2. Click **Actions → Image and templates → Create image**
3. Name: `backend-server-ami`
4. Click **Create image**

### 📋 Step 7.3: Verify AMI Creation
To check progress or confirm success:
1. Go to **EC2 Dashboard → AMIs**
2. Filter by:
   - **Owned by me**
   - Name: `frontend-server-ami` and `backend-server-ami`
3. Wait for Status: **Available**
![Screenshot from 2025-06-25 02-18-28](https://github.com/user-attachments/assets/229d19d3-5ccd-40b9-a93b-61870b645093)

Once both AMIs are in **Available** state, they are ready to be used in:
- Launch templates
- Auto Scaling Groups
- Manual EC2 launches

# Step 8: Configure Launch Templates
### Why use Launch Templates?
- Standardize instance configuration (AMI, instance type, security groups, key pairs, user data scripts).
- Enable Auto Scaling Groups to launch EC2 instances consistently and automatically.
- Allow dynamic customization of instances using user data scripts, so sensitive data like database credentials and endpoints can be injected at launch rather than baked into the AMI.
- Improve flexibility and security by decoupling application code/configuration from the AMI.

# Step 8.1: Create Launch Template for Frontend Servers
### Navigate
- EC2 Dashboard → Launch Templates → **Create launch template**

### Configuration
- **Name:** `frontend-launch-template`
- **AMI:** Select the frontend AMI created in Step 7 (`frontend-server-ami`)
- **Key pair:** Select the key pair created earlier
- **Security Group:** `frontend-server-sg`

### Advanced details → User data
Paste this bash script (used to configure the frontend instance at launch):

```bash
#!/bin/bash
# Replace 'update-me' placeholder in nginx config with the backend ALB DNS name
sed -i 's|update-me|internal-backend-alb-870865928.us-east-1.elb.amazonaws.com|g' /etc/nginx/nginx.conf

# Start nginx service
sudo systemctl start nginx

# Navigate to nginx html directory
cd /usr/share/nginx/html

# Clone the frontend app repository
sudo git clone https://github.com/ajitinamdar-tech/three-tier-architecture-aws.git

# Move frontend files into the html directory
sudo mv /usr/share/nginx/html/three-tier-architecture-aws/frontend/* /usr/share/nginx/html/

# Clean up cloned repo directory
sudo rm -rf /usr/share/nginx/html/three-tier-architecture-aws
```
![Screenshot from 2025-06-25 02-41-58](https://github.com/user-attachments/assets/c3fd113c-fcdc-4003-b411-4702b0f8b60b)

### Explanation:
- **Dynamically sets the backend load balancer’s DNS** in the NGINX configuration so the frontend knows where to send API requests.
- **Starts NGINX** to serve the frontend content.
- **Clones the latest frontend code** on instance boot to ensure the app is always up to date.
- **Keeps the AMI generic** by pushing configuration dynamically via the user data script, allowing flexibility and easier updates.

## Step 8.2: Create Launch Template for Backend Servers
- EC2 Dashboard → Launch Templates → Create launch template

### Configuration:
- **Name:** backend-launch-template
- **AMI:** Select the backend AMI created in Step 7 (`backend-server-ami`)
- **Key pair:** Use the same key pair as above
- **Security Group:** backend-server-sg

### Advanced details → User data
Replace the placeholders below with the actual RDS endpoint, username, and password, then paste this script:
```bash
#!/bin/bash

# Start Apache web server
sudo systemctl start httpd

# Navigate to web root directory
cd /var/www/html

# Clone the application repository
sudo git clone https://github.com/ajitinamdar-tech/three-tier-architecture-aws.git

# Create API directory and move backend API files there
sudo mkdir -p /var/www/html/api
sudo mv /var/www/html/three-tier-architecture-aws/backend/api/* /var/www/html/api/

# Remove cloned repo to keep clean
sudo rm -rf /var/www/html/three-tier-architecture-aws

# Inject database credentials dynamically into the backend PHP app config
sed -i 's/update-me-host/database-1.cjac84smcziu.us-east-1.rds.amazonaws.com/g' /var/www/html/api/db_connection.php
sed -i 's/update-me-username/admin/g' /var/www/html/api/db_connection.php
sed -i 's/update-me-password/<paste-the-password>/g' /var/www/html/api/db_connection.php
```
**Note:**
- Make sure the backend PHP config file (`db_connection.php`) has placeholders `update-me-host`, `update-me-username`, and `update-me-password` exactly matching those in the `sed` commands for these replacements to work properly.
- Adjust the RDS endpoint, username, and password as needed before deploying.

# Explanation of the Backend Launch Script
- Starts Apache so the backend server is ready to serve requests.
- Clones the backend application repository on boot to fetch the latest code.
- Moves the API code into the proper `/var/www/html/api` directory.
- Cleans up the cloned repository after moving the necessary files to keep the server tidy.
- Uses `sed` commands to replace placeholder strings (`update-me-host`, `update-me-username`, `update-me-password`) in the PHP database connection file with actual RDS endpoint, username, and password.

### Final Step: Create the Launch Templates
- Review all settings.
- Click **Create launch template** for both frontend and backend templates.

You can then use these launch templates to:
- Configure Auto Scaling Groups, or
- Manually launch new EC2 instances
with the proper configuration automatically applied.

# Step 9: Create Auto Scaling Groups
## Why use Auto Scaling Groups?
- Automatically manage the number of EC2 instances based on demand (traffic, load, or schedules).
- Maintain high availability by distributing instances across multiple Availability Zones (AZs).
- Replace unhealthy instances automatically to keep the application resilient.
- Scale out (add instances) or scale in (remove instances) dynamically, optimizing costs and performance.

## Step 9.1: Create Auto Scaling Group for Frontend Servers
### Steps:
1. Go to: **EC2 Dashboard → Auto Scaling Groups → Create Auto Scaling Group**
2. Configuration:
   - **Name:** `frontend-server-asg`
   - **Launch Template:** Select the frontend launch template created in Step 8 (`frontend-launch-template`)
   - **Instance Requirements:** Manually add instance types
   - **Instance type:** `t3.micro` (cost-effective, suitable for dev/test)
![Screenshot from 2025-06-25 02-59-52](https://github.com/user-attachments/assets/64c4499b-2db7-4af4-9890-11bd2c258390)

3. Network:
   - **VPC:** select `demo-vpc`
   - **Availability Zones:** select all 3 AZs (e.g., `us-east-1a`, `us-east-1b`, `us-east-1c`)
   - **Subnets:** choose corresponding public web subnets (e.g., `web-public-subnet-1a`, etc.)
![Screenshot from 2025-06-25 03-18-48](https://github.com/user-attachments/assets/d7d020bb-4310-4b9c-b2d5-206d6336d02f)

4. Load Balancer:
   - Attach existing load balancer
   - Select the frontend ALB target group: `frontend-alb-tg`
![Screenshot from 2025-06-25 03-22-30](https://github.com/user-attachments/assets/e38e2097-eb91-400a-bcfa-1909fa64c85a)

5. Configure Group Size and Scaling:
   - **Desired capacity:** 2  
     *(Keeps at least 2 frontend instances running to handle traffic and provide redundancy)*
   - **Minimum capacity:** 2  
     *(ASG won’t scale down below 2 instances, ensuring availability)*
   - **Maximum capacity:** 3  
     *(ASG can scale up to 3 instances during high demand)*

6. Automatic Scaling:
   - Enable Target Tracking Scaling Policy  
     *(Automatically adjusts number of instances to maintain target metrics like CPU usage)*

7. Add Notifications: Skip for now (no notification)

8. Tags:
   - Key: `Name`
   - Value: `frontend-server-ASG`

9. Review and create the Auto Scaling Group

### What this does:
- Automatically launches and terminates EC2 instances based on demand.
- Ensures the frontend layer is highly available across multiple AZs.
- Integrates with the frontend ALB to distribute incoming traffic evenly.
- Supports automatic scaling to optimize costs while handling traffic spikes.
- Provides self-healing by replacing unhealthy instances automatically.

## Step 9.2: Create Auto Scaling Group for Backend Servers

### Steps:
1. Go to: **EC2 Dashboard → Auto Scaling Groups → Create Auto Scaling Group**
2. Configuration:
   - **Name:** `backend-server-asg`
   - **Launch Template:** Select the backend launch template created in Step 8 (`backend-launch-template`)
   - **Instance Requirements:** Manually add instance types
   - **Instance type:** `t3.micro`

3. Network:
   - **VPC:** `demo-vpc`
   - **Availability Zones:** select all 3 AZs (`us-east-1a`, `us-east-1b`, `us-east-1c`)
   - **Subnets:** choose private app subnets (e.g., `app-private-subnet-1a`, etc.)  
     *(Backend servers should be in private subnets for security, receiving traffic only via backend ALB)*
![Screenshot from 2025-06-25 03-31-43](https://github.com/user-attachments/assets/96f41da9-dc50-48cd-bb7d-e83065133910)

4. Load Balancer:
   - Attach existing load balancer
   - Select backend ALB target group: `backend-alb-tg`

5. Configure Group Size and Scaling:
   - **Desired capacity:** 2 (for availability and load handling)
   - **Minimum capacity:** 2
   - **Maximum capacity:** 3

6. Automatic Scaling:
   - Target Tracking Scaling Policy (adjust instances based on load)

7. Add Notifications: None

8. Tags:
   - Key: `Name`
   - Value: `backend-server-ASG`

9. Review and create the Auto Scaling Group

### What this does:
- Automatically manages backend instances to handle business logic and API calls.
- Distributes instances across multiple AZs for fault tolerance.
- Keeps backend servers in private subnets for improved security.
- Automatically scales backend servers based on load, improving responsiveness.
- Integrates with backend ALB to route traffic efficiently.
- Provides resilience and self-healing by replacing unhealthy instances.

# Testing Auto Scaling Groups (ASG) Behavior Upon Creation
- Upon creation, the Auto Scaling Groups (ASGs) will automatically provision **4 EC2 instances**:
  - **2 instances** for the **frontend** ASG (meeting the minimum and desired capacity)
  - **2 instances** for the **backend** ASG (also meeting the minimum and desired capacity)
![Screenshot from 2025-06-25 04-35-55](https://github.com/user-attachments/assets/2729fa22-3d38-4d73-b194-96133ed68b36)

- **User Access:**
  - Users access the service using the **load balancer URL** (which is configured and mapped to a domain name).

## What Will Happen?
1. **Traffic Routing:**
   - The **frontend Application Load Balancer (ALB)** receives user requests on the domain.
   - ALB routes incoming requests evenly across the 2 healthy **frontend EC2 instances**.

2. **Frontend-Backend Communication:**
   - The frontend application makes API requests.
   - These API requests are forwarded by the frontend server to the **backend ALB** URL.
   - The **backend ALB** then distributes these API requests evenly across the 2 healthy **backend EC2 instances**.

3. **High Availability and Load Balancing:**
   - Both frontend and backend ASGs distribute load across multiple AZs, improving fault tolerance.
   - If an instance becomes unhealthy, the ASG automatically replaces it.
   - If traffic increases beyond capacity, the ASG can scale out (up to max capacity) to handle load spikes.

4. **User Experience:**
   - Users experience a resilient, highly available application.
   - Requests are balanced across multiple servers for performance.
   - Backend API responses are reliably served through the backend layer.

---

## Summary
- The ASGs ensure the application runs with the **minimum desired instances** initially.
- The ALBs handle traffic routing and load balancing.
- Auto scaling and self-healing keep the system responsive and available.
- The domain configured to point to the frontend ALB URL enables seamless user access.

---

# Cleanup & Delete Guide
Follow these steps to properly clean up and delete the AWS resources.  
**Note:** Always delete resources in this order to avoid dependency conflicts (e.g., detach Internet Gateway before deleting VPC).

## 1. Delete Auto Scaling Groups  
- Delete **frontend-server-ASG**  
- Delete **backend-server-ASG**

## 2. Delete Launch Templates  
- Delete **frontend-launch-template**  
- Delete **backend-launch-template**

## 3. Terminate EC2 Instances  
- Terminate any running instances manually if not terminated by ASGs.

## 4. Deregister AMIs  
- Deregister **frontend-server-ami**  
- Deregister **backend-server-ami**  
- Optionally delete associated snapshots.

## 5. Delete Load Balancers  
- Delete **frontend-alb**  
- Delete **backend-alb**

## 6. Delete Target Groups  
- Delete **frontend-alb-tg**  
- Delete **backend-alb-tg**

## 7. Delete RDS Database  
- Delete the demo database instance.  
- Optionally delete backups and snapshots.

## 8. Delete RDS Subnet Group  
- Delete **db-subnet-group**

## 9. Delete Security Groups  
- Delete all custom security groups:  
  - **frontend-alb-sg**  
  - **frontend-server-sg**  
  - **backend-alb-sg**  
  - **backend-server-sg**  
  - **database-sg**

## 10. Delete NAT Gateways  
- Delete all NAT gateways created (e.g., **demo-nat-gateway**)

## 11. Detach and Delete Internet Gateway  
- Detach the internet gateway from the VPC.  
- Delete the internet gateway (e.g., **demo-internet-gateway**).

## 12. Delete Route Tables  
- Delete all custom route tables (public and private).

## 13. Delete Subnets  
- Delete all subnets (public, private app, and private DB subnets).

## 14. Delete VPC  
- Delete the **demo-vpc**.

**Important:**  
Deleting in the above order prevents errors related to resource dependencies (e.g., you cannot delete a VPC with attached Internet Gateways or active subnets).

---

# Project Inspiration & Resources

- **Project Inspiration:**  
  [YouTube Video: Three-Tier Architecture on AWS](https://www.youtube.com/watch?v=yeyr6OBMfcE)
- **Source Code and Scripts:**  
  [GitHub Repository: three-tier-architecture-aws](https://github.com/ajitinamdar-tech/three-tier-architecture-aws)

# AWS High-Availability 3-Tier Web Migration Demo

**Portfolio project for AWS Solutions Architect – Associate (SAA-C03) preparation.** This project demonstrates a production-grade migration of a monolithic application to a highly available, scalable, and secure 3-tier architecture on AWS.

---

## 🏗️ Architecture Overview
* **Presentation Tier:** Application Load Balancer (ALB) in Public Subnets.
* **Application Tier:** Auto Scaling Group (ASG) of EC2 instances in **Private Subnets**.
* **Data Tier:** Amazon Aurora PostgreSQL (Multi-AZ Cluster) in isolated Data Subnets.
* **Networking:** Custom VPC with 6 subnets across two Availability Zones (`us-east-1a`, `us-east-1b`).

---

## Phase 1 – VPC & Networking (The Foundation)

- **VPC CIDR:** `10.0.0.0/16`
- **Multi-AZ Design:** Public & private subnets across two AZs for fault tolerance.
- **Egress Strategy:** Used a **Public NAT Gateway** to provide outbound internet access for private instances (required for patching and SSM connectivity).
- **Routing Logic:** - Public Route Table -> Internet Gateway (IGW).
    - Private Route Table -> NAT Gateway.

> ### 💡 Key Lesson: The "NACL Trap"
> During deployment, I encountered a "Silent Timeout" where the SSM Agent could not connect. I identified that the **Network ACL (NACL)** was blocking Outbound HTTPS (Port 443). I resolved this by adding a stateful rule to allow 443, ensuring the private instances could "call home" to AWS APIs.
> During the build, I discovered that placing an instance in a Public Subnet without a Public IP creates a routing loop. I corrected this by moving all application logic to Private Subnets, ensuring the NAT Gateway was the primary egress point.

<image-card alt="VPC Overview" src="screenshots/ha-3tier-vpc-overview.png" ></image-card>
<image-card alt="NAT Gateways" src="screenshots/ha-3tier-nat-gateways.png" ></image-card>

### ⚖️ Architectural Decision: Regional vs. Zonal NAT Gateways

In this deployment, I opted for a **Regional NAT Gateway** architecture (Single NAT GW) rather than a Zonal architecture (NAT GW per AZ).

**The Rationale:**
* **Cost Optimization:** A NAT Gateway costs ~$32.85/month (plus data processing). In a 2-AZ setup, moving from Zonal NATs to a single Regional NAT reduces base networking costs by 50%.
* **Simplicity:** Centralizing egress traffic through one gateway simplifies the monitoring of outbound data.

**The Trade-off (Availability Risk):**
* **Cross-AZ Dependency:** By using a single NAT Gateway in `us-east-1a` for instances in both `1a` and `1b`, the architecture introduces a single point of failure. If Availability Zone `us-east-1a` experiences a service disruption, instances in `us-east-1b` lose internet connectivity (and thus SSM access), even if their own AZ is healthy.
* **Data Transfer Costs:** Traffic moving from a private subnet in `1b` to a NAT in `1a` incurs Inter-AZ data transfer charges ($0.01/GB).

> **Architect's Note:** For a Production environment with strict SLA requirements, I would recommend a **Zonal NAT Gateway** strategy to ensure that the failure of one AZ does not impact the connectivity of others.
---

## Phase 2 – Security & Data Tier

- **Security Group Chaining:** Implemented a "Least Privilege" model where:
    - **ALB SG:** Allows HTTP (80) from `0.0.0.0/0`.
    - **EC2 SG:** Allows HTTP (80) **only** from the ALB Security Group ID.
    - **RDS SG:** Allows PostgreSQL (5432) **only** from the EC2 Security Group ID.
- **Aurora PostgreSQL (Multi-AZ):** Configured for high performance with 6-way replication and automated failover.

<image-card alt="RDS SG Inbound Rule" src="screenshots/security/rds-db-sg-ha-3tier-inbound-rules-screenshot.png" ><img width="1397" height="406" alt="rds-db-sg-ha-3tier-inbound-rules-screenshot" src="https://github.com/user-attachments/assets/a8bd6141-2d07-4906-9eb9-921f6ef923dd" />
</image-card>

---

## Phase 3 – Infrastructure Automation (Launch Templates)

To ensure the application tier is **stateless** and **self-healing**, I developed a Bash bootstrap script used within the EC2 Launch Template.

### 📜 User Data Script Highlights:
- **Automated Stack:** Installs Apache, PHP, and PostgreSQL drivers on boot.
- **Self-Healing Schema:** Logic includes `CREATE TABLE IF NOT EXISTS`,

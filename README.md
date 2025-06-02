# 🚀  **Manara AWS Project - Documentation**

---

## 📋 **Overview**

This project implements a multi-tier AWS infrastructure with the following components:
- VPC with public and private subnets across two availability zones
- Load balancers for traffic distribution
- Proxy servers in public subnets
- Backend web servers in private subnets
- Security groups for access control
- Remote state management with S3 and DynamoDB
- CloudWatch & SNS
- IAM
- Amazon RDS
- Auto Scaling Group (ASG): Ensures instances scale based on demand.
---
## 🏗️ **Architecture Diagram**
![Untitled Diagram drawio (1)](https://github.com/user-attachments/assets/ffe43373-fa7e-4586-a2a6-63eb84553ab4)


---

## 🧩 **Architecture Components**

| Component                    | Description                                                            |
|------------------------------|------------------------------------------------------------------------|
| **VPC**                      | Isolated AWS network environment.                                      |
| **Internet Gateway (IGW)**   | Connects VPC to the internet; enables public subnet connectivity.      |
| **Public Subnets**           | Host internet-facing resources (e.g., Bastion Host).                   |
| **Private Subnets**          | Host internal resources like application servers.                      |
| **Route Tables**             | Routes traffic appropriately (public → IGW, private → NAT Gateway).    |
| **NAT Gateway**              | Allows private subnet instances outbound internet access securely.     | 
| **Elastic IP**               | Static IP assigned to NAT Gateway for stable connectivity.             |
| **Security Groups**          | Virtual firewalls controlling inbound/outbound traffic.                |
| **Nginx Instances**          | Reverse proxy servers located in public subnets.                       |
| **Apache Backend Instances** | Web servers located in private subnets.                                |
| **Load Balancer**            | Distributes incoming traffic for fault tolerance and high availability.|
| **CloudWatch & SNS**         | Monitor performance and send alerts.                                   |
| **IAM**                      | Role-based access to instances.                                        |
| **Amazon RDS**               | Stores and manages the application's data.                             |
| **Auto Scaling Group(ASG)**  | Ensures instances scale based on demand.                               |

---

## 🚀 **Network Traffic Flow**
1. User requests originate from the Internet.

2. Traffic hits the entry-point Load Balancer (e.g., NLB).

3. The entry-point LB distributes traffic across AZs, forwarding it to the internal Load
Balancer (e.g., ALB) or potentially the proxy layer.

4. (If proxies are intermediate) Proxies in the public subnets receive traffic, perform
their functions, and forward requests to the internal ALB.

5. The internal ALB distributes traffic across the healthy Backend Web Server (BE WS)
instances located in the private subnets of both AZs.

6. BE WS instances process the requests, interacting with the Amazon RDS database
in the private subnet as needed.

7. BE WS instances may also interact with other services like Email or Monitoring.

8. Responses flow back through the same path to the user.

---

# **High Availability and Scalability**

# *High Availability*:

Achieved by deploying critical components (Load Balancers,
Proxies, BE WS) across two Availability Zones. If one AZ fails, the resources in the
other AZ can continue serving traffic. A Multi-AZ RDS deployment (recommended)
further enhances database availability.

# *Scalability*:
The Backend Application Layer (BE WS) utilizes an Auto Scaling group.
This allows the application to automatically scale out (add instances) during
periods of high traffic and scale in (remove instances) during low traffic periods,
ensuring performance and cost-efficiency.
---
## 🔐 **Security Architecture**

- Separate **Security Groups** for:
  - Proxy instances (open to public)
  - Backend instances (only accessible by ALB)
  - RDS (only accessible by backend instances)
- **NACLs** and **Routing Tables** properly configured to isolate tiers
- **No direct public access** to backend or database layers

---
## **Step-by-Step Deployment**
### 1. VPC Setup

First, we establish the network foundation. You can use the default VPC or create a new one as detailed below.

*   **Create VPC:**
    *   Navigate to the VPC service in the AWS Management Console.
    *   Create a new VPC.
    *   **Name:** `manara-aws-project-dev-vpc`
    *   **IPv4 CIDR block:** `10.0.0.0/16` 
*   **Create Subnets:** Create four subnets across two Availability Zones (AZs) for high availability.
    *   **Public Subnet 1 (AZ1):**
        *   Name: `manara-aws-project-dev-public-subnet-1`
        *   CIDR: `10.0.0.0/24`
    *   **Public Subnet 2 (AZ2):**
        *   Name: `manara-aws-project-dev-public-subnet-2`
        *   CIDR: `10.0.2.0/24`
    *   **Private Subnet 1 (AZ1):**
        *   Name: `manara-aws-project-dev-private-subnet-1`
        *   CIDR: `10.0.1.0/24`
    *   **Private Subnet 2 (AZ2):**
        *   Name: `manara-aws-project-dev-private-subnet-2`
        *   CIDR: `10.0.3.0/24`
*   **Internet Gateway (IGW):**
    *   Create an Internet Gateway.
    *   Name: `manara-aws-project-dev-igw`
    *   Attach it to your `manara-aws-project-dev-vpc`.
*   **Route Tables:**
    *   **Public Route Table:** Create a route table for public subnets.
        *   Name: `manara-aws-project-dev-public-route-table`
        *   Add a route: Destination `0.0.0.0/0`, Target `manara-aws-project-dev-igw`.
        *   Associate this route table with `manara-aws-project-dev-public-subnet-1`
             and `manara-aws-project-dev-public-subnet-2`.
    *   **Private Route Table:** Create a route table for private subnets.
        *   Name: `manara-aws-project-dev-private-route-table`
        *   (Optional but recommended for outbound access): Create NAT Gateways in each public subnet and add routes in the private route table pointing `0.0.0.0/0` to the respective NAT Gateway in the same AZ.
        *   Associate this route table with `manara-aws-project-dev-private-subnet-1`
            and `manara-aws-project-dev-private-subnet-2`.

### 2. Security Groups

Security groups act as virtual firewalls. We need groups for the Application Load Balancer (ALB), the EC2 instances, and the RDS database.

*   **ALB Security Group:**
    *   Name: `ALB-SecurityGroup`
    *   Description: Controls traffic to the ALB.
    *   VPC: `ScalableAppVPC-vpc`
    *   **Inbound Rules:**
        *   Allow HTTP (Port 80) from Anywhere (`0.0.0.0/0`, `::/0`)
        *   Allow HTTPS (Port 443) from Anywhere (`0.0.0.0/0`, `::/0`)
*   **EC2 Instance Security Group:**
    *   Name: `EC2-SecurityGroup`
    *   Description: Controls traffic to the web server instances.
    *   VPC: `ScalableAppVPC-vpc`
    *   **Inbound Rules:**
        *   Allow HTTP (Port 80) from `ALB-SecurityGroup` (Source: Select the ALB SG ID).
        *   Allow HTTPS (Port 443) from `ALB-SecurityGroup` (Source: Select the ALB SG ID).
        *   Allow SSH (Port 22) from `Your Public IP Address` (e.g., `x.x.x.x/32`).
*   **RDS Security Group:**
    *   Name: `RDS-SecurityGroup`
    *   Description: Controls traffic to the RDS database.
    *   VPC: `ScalableAppVPC-vpc`
    *   **Inbound Rules:**
        *   Allow `MySQL/Aurora` (Port 3306) or `PostgreSQL` (Port 5432) from `EC2-SecurityGroup` (Source: Select the EC2 SG ID).


### 3. IAM Role for EC2

Create an IAM role that EC2 instances can assume to interact with other AWS services securely.

*   Navigate to IAM > Roles > Create role.
*   **Trusted entity type:** AWS service
*   **Use case:** EC2
*   **Attach Permissions Policies:**
    *   `AmazonEC2ReadOnlyAccess`
    *   `CloudWatchAgentServerPolicy`
    *   `AmazonSSMManagedInstanceCore`
    *   `AmazonS3ReadOnlyAccess` (Note: `AmazonS3Role` is not a standard policy; using `AmazonS3ReadOnlyAccess` as a common requirement. Adjust if specific S3 write access is needed).
*   **Role Name:** `EC2-App-Role`
*   Create the role.

### 4. Launch Template

A Launch Template specifies the configuration for EC2 instances launched by the Auto Scaling group.

*   Navigate to EC2 > Launch Templates > Create launch template.
*   **Launch template name:** `web-app-template`
*   **Template version description:** Initial version
*   **Application and OS Images (AMI):** Select an Amazon Linux 2 or Ubuntu Server AMI.
*   **Instance type:** `t2.micro` (or another suitable type).
*   **Key pair (login):** Select an existing key pair or create a new one if you need SSH access.
*   **Network settings:**
    *   **Subnet:** Do not include in launch template (will be specified by ASG).
    *   **Security groups:** Select the `EC2-SecurityGroup` created earlier.
*   **Advanced details:**
    *   **IAM instance profile:** Select the `EC2-App-Role` created earlier.
    *   **User data:** Paste the following script to install a basic web server:

```bash
#!/bin/bash
    yum update -y
    yum install -y httpd -y
    systemctl start httpd
    systemctl enable httpd

    cat <<EOT > /var/www/html/index.html
    <!DOCTYPE html>
    <html lang="en">
    <head>
      <meta charset="UTF-8">
      <meta name="viewport" content="width=device-width, initial-scale=1.0">
      <title>NTI Backend Server</title>
      <link rel="preconnect" href="https://fonts.googleapis.com">
      <link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
      <link href="https://fonts.googleapis.com/css2?family=Montserrat:wght@300;400;500;600;700&display=swap" rel="stylesheet">
      <style>
        :root {
          --primary: #f37021;
          --secondary: #6c757d;
          --success: #28a745;
          --light: #f8f9fa;
          --dark: #343a40;
          --white: #ffffff;
          --shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
          --radius: 8px;
        }
        
        * {
          margin: 0;
          padding: 0;
          box-sizing: border-box;
          font-family: 'Montserrat', sans-serif;
        }
        
        body {
          background: #f5f7fa;
          color: var(--secondary);
          min-height: 100vh;
          display: flex;
          flex-direction: column;
          align-items: center;
          justify-content: center;
          padding: 20px;
        }
        
        .container {
          max-width: 800px;
          width: 100%;
          text-align: center;
          overflow: hidden;
          position: relative;
        }
        
        .nti-logo-container {
          margin-bottom: 40px;
          position: relative;
          display: inline-block;
        }
        
        .nti-logo-img {
          max-width: 300px;
          height: auto;
          margin-bottom: 20px;
        }
        
        .nti-full-name {
          position: relative;
          font-size: 18px;
          font-weight: 600;
          color: var(--secondary);
          letter-spacing: 1px;
          margin-top: 15px;
        }
        
        h1 {
          font-size: 32px;
          margin-bottom: 40px;
          color: var(--primary);
          font-weight: 600;
        }
        
        .server-info {
          background-color: var(--light);
          padding: 30px;
          border-radius: 15px;
          text-align: left;
          border-left: 5px solid var(--primary);
          box-shadow: 0 5px 15px rgba(0, 0, 0, 0.03);
          margin-bottom: 40px;
        }
        
        .server-info h2 {
          font-size: 24px;
          margin-bottom: 25px;
          color: var(--primary);
          font-weight: 600;
          display: flex;
          align-items: center;
        }
        
        .server-info h2::before {
          content: "⚙️";
          margin-right: 10px;
          font-size: 28px;
        }
        
        .server-info p {
          margin-bottom: 20px;
          font-size: 16px;
          line-height: 1.6;
          display: flex;
          align-items: center;
        }
        
        .server-info p strong {
          min-width: 140px;
          display: inline-block;
          color: #555;
        }
        
        .server-info code {
          background-color: #e9ecef;
          padding: 8px 12px;
          border-radius: 5px;
          font-family: "Courier New", monospace;
          color: var(--dark);
          font-size: 15px;
          order-left: 3px solid var(--primary);
          flex: 1;
        }
        
        .credits {
          margin-top: 30px;
          text-align: center;
        }
        
        .credits p {
          font-size: 16px;
          margin-bottom: 10px;
          color: var(--secondary);
          font-weight: 500;
        }
        
        .footer {
          margin-top: 40px;
          font-size: 14px;
          color: #888;
          position: relative;
          padding-top: 20px;
        }
        
        .footer::before {
          content: "";
          position: absolute;
          top: 0;
          left: 25%;
          width: 50%;
          height: 1px;
          background: #ddd;
        }
        
        @media (max-width: 768px) {
          .nti-logo-img {
            max-width: 200px;
          }
          
          h1 {
            font-size: 24px;
          }
          
          .server-info p strong {
            min-width: 120px;
          }
        }
      </style>
    </head>
    <body>
        <div class="container">
            <div class="nti-logo-container">
                <img src="https://pages.manara.tech/hubfs/01-manara-black-orange-3.png" alt="NTI Logo" class="nti-logo-img">
                <div class="nti-full-name">Manara-Tech</div>
            </div>
            <h1>Manara Backend Server</h1>
            
            <div class="server-info">
                <h2>Server Information</h2>
                <p><strong>Hostname:</strong> <code>$(hostname)</code></p>
                <p><strong>Region:</strong> <code>us-east-1</code></p>
                <p><strong>Availability Zone:</strong> <code>us-east-1b</code></p>
            </div>
            
            <div class="credits">
                <p>Developed by: Mahmoud Yassen</p>
            </div>
            
            <div class="footer">
                &copy; 2025 Manara AWS Infrastructure Project
            </div>
        </div>
    </body>
    </html>
    EOT

    systemctl restart httpd
  EOF
```
*   Create the launch template.
### 5. Create Application Load Balancer (ALB)

The ALB distributes incoming traffic across your EC2 instances.

*   Navigate to EC2 > Load Balancers > Create Load Balancer.
*   **Select:** Application Load Balancer.
*   **Basic configuration:**
    *   **Load balancer name:** `web-ALB`
    *   **Scheme:** Internet-facing
    *   **IP address type:** IPv4
*   **Network mapping:**
    *   **VPC:** `ScalableAppVPC-vpc`
    *   **Mappings:** Select both Availability Zones used for your public subnets and choose `public-subnet-az1` and `public-subnet-az2`.
*   **Security groups:** Select the `ALB-SecurityGroup`.
*   **Listeners and routing:**
    *   **Listener:** Protocol `HTTP`, Port `80`.
    *   **Default action:** Forward to a target group.
    *   **Create target group:**
        *   Target type: `Instances`
        *   Target group name: `web-app-tg`
        *   Protocol: `HTTP`, Port `80`
        *   VPC: `ScalableAppVPC-vpc`
        *   Health checks: Protocol `HTTP`, Path `/` (or your application's health check path).
        *   Register targets: Skip this for now; the ASG will register instances.
        *   Create the target group.
    *   Go back to the ALB creation page and select the newly created `web-app-tg` for the listener's default action.
*   Create the load balancer.

### 6. Create Auto Scaling Group (ASG)

The ASG automatically adjusts the number of EC2 instances based on demand or schedule.

*   Navigate to EC2 > Auto Scaling Groups > Create Auto Scaling group.
*   **Choose launch template or configuration:**
    *   **Auto Scaling group name:** `web-ASG`
    *   **Launch template:** Select `web-app-template`, Version `Latest`.
*   **Choose instance launch options:**
    *   **Network:**
        *   VPC: `ScalableAppVPC-vpc`
        *   **Availability Zones and subnets:** Select `public-subnet-az1` and `public-subnet-az2`.
*   **Configure advanced options:**
    *   **Load balancing:** Attach to an existing load balancer.
    *   Choose from your load balancer target groups: Select `web-app-tg`.
    *   Health checks: Enable ELB health checks (optional but recommended).
*   **Configure group size and scaling policies:**
    *   **Desired capacity:** `2`
    *   **Minimum capacity:** `2`
    *   **Maximum capacity:** `5`
    *   **Scaling policies:** Choose `Target tracking scaling policy`.
        *   Metric type: `Average CPU utilization`
        *   Target value: `80` (Scale-out when CPU > 80%)
        *   (Optionally add a scale-in policy, e.g., CPU < 40%)
*   Review and create the Auto Scaling group.

### 7. CloudWatch Alarm (for Scaling)

The ASG scaling policy created in the previous step automatically creates the necessary CloudWatch alarms. You can view and customize them further if needed.

*   Navigate to CloudWatch > Alarms > All alarms.
*   You should see alarms named similar to `TargetTracking-web-ASG-...-AlarmHigh-...` and potentially `...AlarmLow...`.
*   (Optional) If you need a separate alarm for notifications (e.g., via SNS), you can create one manually:
    *   Click `Create alarm`.
    *   Select metric > EC2 > By Auto Scaling Group > Select `web-ASG` > `CPUUtilization`.
    *   Conditions: Static, Greater/Equal, Threshold `80`.
    *   Actions: Configure an SNS topic for notification.
    *   Alarm name: `CPU-High-Notification-Alarm` (or similar).

### 8. (Optional) RDS - Multi-AZ Database

Set up a managed relational database if your application requires one.

*   Navigate to RDS > Databases > Create database.
*   **Choose a database creation method:** Standard Create.
*   **Engine options:** Select `MySQL` or `PostgreSQL`.
*   **Templates:** Choose `Production` (enables Multi-AZ and other best practices by default) or `Dev/Test`.
*   **Settings:**
    *   **DB instance identifier:** `webapp-db`
    *   Set master username and password.
*   **Instance configuration:** Choose an appropriate DB instance class.
*   **Storage:** Configure storage type and size.
*   **Availability & durability:**
    *   **Multi-AZ deployment:** Ensure `Create a standby instance` (or equivalent Multi-AZ option) is selected for high availability.
*   **Connectivity:**
    *   **Virtual private cloud (VPC):** Select `ScalableAppVPC-vpc`.
    *   **Subnet group:** Create a new DB subnet group including `private-subnet-az1` and `private-subnet-az2`.
    *   **Public access:** Select `No`.
    *   **VPC security group (firewall):** Choose existing > Select `RDS-SecurityGroup`.
*   Configure other settings (database options, backup, monitoring, maintenance) as needed.
*   Create database.
*   **Connection:** Once the database is available, find its **Endpoint** on the Connectivity & security tab. Use this endpoint in your application's configuration (running on the EC2 instances) to connect to the database. Ensure the EC2 instances' security group (`EC2-SecurityGroup`) allows outbound traffic to the RDS security group (`RDS-SecurityGroup`) on the database port if not already covered by default outbound rules.

#  TASK 1: Highly Available Web Application with Auto Scaling (AWS)
---
#  Architecture Overview

```
                         Internet
                             │
                             │  HTTP (80)
                             ▼
                ┌───────────────────────────┐
                │   Application Load        │
                │     Balancer (ALB)        │
                │  (Security Group: Allow   │
                │   HTTP from Internet)     │
                └─────────────┬─────────────┘
                              │
                ──────────────┼──────────────
                              │
                 Routes traffic to Target Group
                              │
        ┌─────────────────────┴─────────────────────┐
        │                                           │
        ▼                                           ▼
 ┌─────────────────┐                       ┌─────────────────┐
 │  Availability   │                       │  Availability   │
 │   Zone A        │                       │   Zone B        │
 │                 │                       │                 │
 │  EC2 Instance 1 │                       │  EC2 Instance 2 │
 │  (Custom AMI)   │                       │  (Custom AMI)   │
 │  Apache + App   │                       │  Apache + App   │
 │  EBS Volume     │                       │  EBS Volume     │
 │                 │                       │                 │
 └─────────────────┘                       └─────────────────┘
        ▲                                           ▲
        │                                           │
        └──────────── Auto Scaling Group ───────────┘
                    (Min:2  Max:4  Desired:2)
                    
```

---

#  Security Group Flow

```
Internet
   │
   ▼
ALB Security Group
   - Allow HTTP (80) from 0.0.0.0/0
   │
   ▼
EC2 Security Group
   - Allow HTTP (80) from ALB SG only
   - Allow SSH (22) from My IP only
```

---

#  Components Explanation

###  Application Load Balancer

* Deployed in **2 Availability Zones**
* Performs health checks (`/`)
* Distributes traffic evenly

###  Auto Scaling Group

* Minimum: 2
* Maximum: 4
* Scale out → CPU ≥ 70%
* Scale in → CPU ≤ 30%
* Self-healing (replaces unhealthy instances)

###  EC2 Instances

* Created using **Launch Template**
* Use **Custom AMI**
* Each instance has its own **EBS volume**
* Web server installed (Apache)

---

#  STEP 1: Create a Base EC2 Instance (For Custom AMI)

### 1️ Launch EC2 Instance

* Go to **EC2 Console**
* Click **Launch Instance**
* Choose: Amazon Linux 2
* Instance type: t2.micro (lab purpose)
* Attach EBS (default root volume is fine)
* <img width="1911" height="822" alt="Screenshot 2026-02-25 161504" src="https://github.com/user-attachments/assets/f528b515-b3e7-43c1-ab80-f300ce944e68" />
* <img width="1891" height="825" alt="Screenshot 2026-02-25 163016" src="https://github.com/user-attachments/assets/09010974-2b61-4a7c-957a-5e2bb059298b" />

### 2️ Configure Security Group

Allow:

* HTTP (80) → From Anywhere (temporary for setup)
* SSH (22) → Your IP only
* <img width="1910" height="830" alt="Screenshot 2026-02-25 164745" src="https://github.com/user-attachments/assets/83181cc1-6f93-4a69-ae00-fb3fa07fb8b3" />
<img width="1917" height="837" alt="Screenshot 2026-02-25 165012" src="https://github.com/user-attachments/assets/3ee6df37-ff4d-48ff-a8d5-b48b73db4a73" />
<img width="1905" height="805" alt="Screenshot 2026-02-25 165237" src="https://github.com/user-attachments/assets/cc18d3c0-5a6b-49a2-a379-643e102f917f" />


### 3️ Install Web Server

SSH into instance:

```bash
sudo yum update -y
sudo yum install httpd -y
sudo systemctl start httpd
sudo systemctl enable httpd
```

Create test page:

```bash
echo "Hello from $(hostname)" | sudo tee /var/www/html/index.html
```

Test:
Open public IP → You should see hostname message.

---

#  STEP 2: Create Custom AMI

1. Select EC2 instance
2. Click **Actions → Image and Templates → Create Image**
3. Name it: `WebApp-Custom-AMI`
4. Create Image

Wait until AMI status = **Available**
<img width="1912" height="821" alt="Screenshot 2026-02-25 163105" src="https://github.com/user-attachments/assets/5a7ae1d9-39cb-4835-b3a0-e55575296244" />

---

#  STEP 3: Create Launch Template

Go to:
EC2 → Launch Templates → Create Launch Template

### Configure:

* AMI → Select your Custom AMI
* Instance type → t2.micro
* Key pair → Select yours
* Security Group → Create new one:
* <img width="1898" height="822" alt="Screenshot 2026-02-25 163910" src="https://github.com/user-attachments/assets/0f35ed9e-f780-4463-b805-7fa869a3e6fd" />
<img width="1907" height="758" alt="Screenshot 2026-02-25 163925" src="https://github.com/user-attachments/assets/b78da9bb-d33c-4460-a44e-3ced3c86254b" />


### EC2 Security Group Rules:

Inbound:

* HTTP (80) → From ALB Security Group only
* SSH (22) → Your IP only

Click **Create Launch Template**

---

#  STEP 4: Create Target Group

Go to:
EC2 → Target Groups → Create Target Group

### Settings:

* Target type: Instances
* Protocol: HTTP
* Port: 80
* VPC: Default or your lab VPC

### Health Check:

* Path: /
* Healthy threshold: 2
* Unhealthy threshold: 2
* Interval: 30 sec

Create Target Group

---

#  STEP 5: Create Application Load Balancer (ALB)

Go to:
EC2 → Load Balancers → Create Load Balancer → Application Load Balancer

### Configure:

* Scheme: Internet-facing
* Select 2 Availability Zones
* Security Group (ALB SG):

Inbound:

* HTTP (80) → Anywhere (0.0.0.0/0)

### Listener:

* Forward to Target Group (created earlier)

Create Load Balancer

---

#  STEP 6: Create Auto Scaling Group (ASG)

Go to:
EC2 → Auto Scaling Groups → Create ASG

### Configuration:

* Select Launch Template
* Choose VPC
* Select **2 Availability Zones**
* Attach to existing Target Group
* Health check type:

  * EC2 + ELB

### Group Size:

* Desired: 2
* Minimum: 2
* Maximum: 4

Create ASG

---

#  STEP 7: Configure Scaling Policies

Inside ASG → Automatic Scaling

### Create Scaling Policy

##  Scale Out Policy

* Type: Target tracking
* Metric: Average CPU Utilization
* Target value: 70%

OR Simple scaling:

* Add CloudWatch Alarm:

  * CPU ≥ 70%
  * Add 1 instance

---

##  Scale In Policy

* CPU ≤ 30%
* Remove 1 instance

---

#  STEP 8: Test Scaling

### Generate CPU Load

SSH into instance:

```bash
sudo yum install stress -y
stress --cpu 2 --timeout 300
```

Monitor:

* CloudWatch → CPU Utilization
* Watch ASG launch new instance

You should see:
Instance count increase (Scale Out event)

---

#  Security Group Layering (Very Important)

### ALB Security Group:

* Allow HTTP from Internet

### EC2 Security Group:

* Allow HTTP only from ALB SG
* Allow SSH only from your IP

This ensures proper layered security.

---

##  Why ALB not NLB?

| ALB                   | NLB                     |
| --------------------- | ----------------------- |
| Layer 7 (HTTP/HTTPS)  | Layer 4 (TCP/UDP)       |
| Path-based routing    | No path routing         |
| Health checks on HTTP | Basic TCP health checks |

We use **ALB** because:

* It supports HTTP-based routing
* Better health checks
* Ideal for web applications

---

##  Why Launch Template instead of Launch Configuration?

Launch Template:

* Supports latest EC2 features
* Versioning support
* More flexible
* Required for modern ASG setups

Launch Configuration:

* Legacy (older method)
* No versioning
* Limited features

---


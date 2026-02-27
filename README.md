#  TASK 1: Highly Available Web Application with Auto Scaling (AWS)

### This project demonstrates the design and implementation of a highly available and production-ready web application infrastructure on Amazon EC2 within Amazon Web Services.

### The objective is to build a fault-tolerant system capable of handling unpredictable traffic spikes without downtime. To achieve this, the architecture leverages a Multi-Availability Zone deployment strategy combined with Auto Scaling and Layer 7 load balancing.
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
 <img width="1910" height="830" alt="Screenshot 2026-02-25 164745" src="https://github.com/user-attachments/assets/83181cc1-6f93-4a69-ae00-fb3fa07fb8b3" />
<img width="1917" height="837" alt="Screenshot 2026-02-25 165012" src="https://github.com/user-attachments/assets/3ee6df37-ff4d-48ff-a8d5-b48b73db4a73" />
<img width="1905" height="805" alt="Screenshot 2026-02-25 165237" src="https://github.com/user-attachments/assets/cc18d3c0-5a6b-49a2-a379-643e102f917f" />

### 3️ User data:
#!/bin/bash
yum update -y
yum install httpd -y
systemctl start httpd
systemctl enable httpd
echo "<h1>HTML + CSS + JavaScript code</h1>" > /var/www/html/index.html
Test:
Open public IP 
<img width="1919" height="949" alt="Screenshot 2026-02-26 104011" src="https://github.com/user-attachments/assets/a538d541-bf8a-4186-94ea-c67fc88346ec" />

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
* <img width="1910" height="830" alt="Screenshot 2026-02-25 164745" src="https://github.com/user-attachments/assets/13e461fc-f085-4b85-80c5-fbbeec24d77f" />
* <img width="1917" height="837" alt="Screenshot 2026-02-25 165012" src="https://github.com/user-attachments/assets/965881cf-3616-4b03-a9ee-35d4579d7945" />
<img width="1905" height="805" alt="Screenshot 2026-02-25 165237" src="https://github.com/user-attachments/assets/60b74a78-7c6b-4572-95c1-1e73ec87f98e" />

Click **Create Launch Template**
<img width="1905" height="812" alt="Screenshot 2026-02-25 165735" src="https://github.com/user-attachments/assets/571042c0-d251-4631-ba8f-ae748963d91b" />

---

#  STEP 4: Create Target Group

Go to:
EC2 → Target Groups → Create Target Group

### Settings:

* Target type: Instances
* Protocol: HTTP
* Port: 80
* VPC: Default or your lab VPC
* <img width="1919" height="765" alt="Screenshot 2026-02-25 165850" src="https://github.com/user-attachments/assets/c4a78df1-d07a-45dd-8eb1-15eaddcb1b8d" />
<img width="1910" height="748" alt="Screenshot 2026-02-25 165918" src="https://github.com/user-attachments/assets/383df78b-648f-42b8-ab2a-4a30366c7c52" />

### Health Check:

* Path: /
* Healthy threshold: 2
* Unhealthy threshold: 2
* Interval: 30 sec

Create Target Group
<img width="1910" height="754" alt="Screenshot 2026-02-25 170426" src="https://github.com/user-attachments/assets/d40ed3f6-8cca-4130-ad3b-3a1120feebad" />

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
<img width="1913" height="752" alt="Screenshot 2026-02-26 104232" src="https://github.com/user-attachments/assets/5d8333f1-2bfd-4651-9260-4772cfbd7bf1" />

### Listener:

* Forward to Target Group (created earlier)

Create Load Balancer
<img width="1918" height="831" alt="Screenshot 2026-02-25 173850" src="https://github.com/user-attachments/assets/cc36e067-be73-4a2f-be34-cf69850ef6d1" />


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
  * <img width="1919" height="797" alt="Screenshot 2026-02-25 181433" src="https://github.com/user-attachments/assets/37ba9aba-6712-4218-bce7-8a518ce6580b" />
<img width="1900" height="830" alt="Screenshot 2026-02-25 181447" src="https://github.com/user-attachments/assets/46d798d3-fc27-434e-945d-86632428d8f5" />
<img width="1889" height="745" alt="Screenshot 2026-02-25 181527" src="https://github.com/user-attachments/assets/17cea2dd-1d83-420f-a5f9-92a802f5b1a0" />
<img width="1902" height="819" alt="Screenshot 2026-02-25 181858" src="https://github.com/user-attachments/assets/a80a3c80-08a0-4e47-9739-70efc9a3fceb" />

### Group Size:

* Desired: 2
* Minimum: 2
* Maximum: 4

Create ASG
<img width="1898" height="828" alt="Screenshot 2026-02-25 182202" src="https://github.com/user-attachments/assets/dead527a-7cbb-456b-ba3b-1e29bc32450a" />

---

#  STEP 7: Configure Scaling Policies

Inside ASG → Automatic Scaling

### Create Scaling Policy

##  Scale Out Policy

* Type: Target tracking
* Metric: Average CPU Utilization
* Target value: 70%
* 

OR Simple scaling:

* Add CloudWatch Alarm:

  * CPU ≥ 70%
  * Add 1 instance
  * <img width="1918" height="771" alt="Screenshot 2026-02-26 120351" src="https://github.com/user-attachments/assets/d27946df-b563-46cb-b2b4-e7c37d501656" />
<img width="1919" height="708" alt="Screenshot 2026-02-26 120401" src="https://github.com/user-attachments/assets/d22b3500-08f4-4b6f-b30f-619628e40fba" />
<img width="1919" height="798" alt="Screenshot 2026-02-26 120735" src="https://github.com/user-attachments/assets/4e7ff7ce-1034-4e39-ad3e-22417658b9b9" />

---

##  Scale In Policy

* CPU ≤ 30%
* Remove 1 instance
<img width="1913" height="770" alt="Screenshot 2026-02-26 121643" src="https://github.com/user-attachments/assets/3e500e7f-ed37-4fe5-b9b2-81b4f7d33e88" />
<img width="1919" height="785" alt="Screenshot 2026-02-26 121653" src="https://github.com/user-attachments/assets/c492038c-4e75-431b-9742-6cbb6e41745e" />
<img width="1919" height="767" alt="Screenshot 2026-02-26 121918" src="https://github.com/user-attachments/assets/c40371cf-e8aa-4618-903a-e0428029874b" />
<img width="1913" height="648" alt="Screenshot 2026-02-26 122435" src="https://github.com/user-attachments/assets/5fb6ebf8-62ce-4681-830c-93b65c3101f5" />

---

#  STEP 8: Test Scaling:

<img width="1918" height="790" alt="Screenshot 2026-02-26 123322" src="https://github.com/user-attachments/assets/d50420e9-6380-4b18-94b3-0ef51c3d98ac" />
<img width="1676" height="791" alt="Screenshot 2026-02-27 102200" src="https://github.com/user-attachments/assets/cdd53f53-19f7-46ed-a971-e2ccc84e8eb5" />

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


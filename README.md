# Automated SNS Alerts for EC2 Disk Space Utilization
### Assignment 16 — AWS Lambda | CloudWatch | SNS | EventBridge

---

## 📌 Objective
Implement an automated alerting system that monitors **EC2 disk space utilization** using AWS Lambda.  
If disk usage exceeds **85%**, the system triggers an **SNS Email Notification**.  
The Lambda function runs **daily** using an **EventBridge Scheduled Rule**.

---

## 🏗 Architecture

```mermaid
flowchart LR
A[EC2 Instance] --> B[CloudWatch Agent]
B --> C[CloudWatch Metrics]
C --> D[Lambda Function]
D --> E[SNS Topic]
E --> F[Email Notification]
D <-->|Daily Trigger| G[EventBridge Rule]

## Components Used

| AWS Service           | Purpose                        |
| --------------------- | ------------------------------ |
| EC2 Instance (Ubuntu) | Server to monitor              |
| CloudWatch Agent      | Collects disk metrics          |
| CloudWatch Metrics    | Stores disk utilization metric |
| Lambda                | Reads metrics & sends alerts   |
| SNS Topic             | Sends notification emails      |
| EventBridge           | Schedules Lambda execution     |
| IAM Roles             | Permissions for EC2 & Lambda   |

## 1. EC2 Instance (Monitored Server)
Shows the Ubuntu EC2 instance (t2.micro) with the IAM role attached (e.g. Nikitha-EC2-CloudWatchAgentRole).

<img width="1280" height="692" alt="EC2 Instance Summary" src="https://github.com/user-attachments/assets/3b4d548c-69b8-4c16-bb0d-9eec7c2497df" />

<img width="1156" height="536" alt="image" src="https://github.com/user-attachments/assets/ae9c7991-43ee-441f-886e-b2d3685cdac1" />


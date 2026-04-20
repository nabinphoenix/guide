# A3-G7 AWS Infrastructure: Testing & Validation Manual
**Project:** CT097-3-3-CSVC Cloud Infrastructure and Services
**Environment:** AWS us-east-1 (Production-Ready Demo)

---

## 📑 Table of Contents
1. [Setup: Visual Verification](#setup-visual-verification)
2. [Module 1: Security Validation](#module-1-security-validation)
3. [Module 2: High Availability (HA)](#module-2-high-availability-ha)
4. [Module 3: Scalability & Performance](#module-3-scalability--performance)
5. [Module 4: Application & Data Integrity](#module-4-application--data-integrity)
6. [Appendix: Execution Commands](#appendix-execution-commands)

---

## 🛠️ Setup: Visual Verification
Before beginning the formal validation, follow these steps to enable **Server Identification** on the website. This allows you to "see" which EC2 instance is serving your request.

| Task | Action | Command/Details |
|---|---|---|
| **Step 1** | Connect to EC2 | Use **EC2 Instance Connect** for both `10.0.3.177` and `10.0.3.18`. |
| **Step 2** | Navigate to Web Root | `cd /var/www/html` |
| **Step 3** | Edit index.php | `sudo nano index.php` |
| **Step 4** | Add Info Block | Replace the Copyright div with the PHP snippet below. |

### Visual Identification Snippet
Replace the footer in `index.php` with this code:
```php
<div id="Copyright" class="center">
    <h5>&copy; 2024, A3-G7 CSVC Team. All rights reserved.</h5>
    <div style="margin-top: 10px; color: #555; background: #f0f0f0; padding: 10px; border-radius: 8px; display: inline-block; border: 1px dotted #0073bb; font-family: monospace;">
        <strong>📍 Active Node:</strong> <?php echo $_SERVER['SERVER_ADDR']; ?>
    </div>
</div>
```

---

## 🛡️ Module 1: Security Validation
**Objective:** Confirm isolation of the Three-Tier architecture and adherence to the Principle of Least Privilege.

| ID | Test Category | Procedure | Expected Result | Status |
|:---:|---|---|---|:---:|
| **SEC-01** | **Host Isolation** | Attempt to SSH into Private EC2 using a Public IP. | Connection Timeout (No Public IP/Direct Access). | [ ] |
| **SEC-02** | **ALB Gateway** | Access app via `http://a3-g7-csvc-alb...` DNS. | Website loads; ALB acts as the sole entry point. | [ ] |
| **SEC-03** | **DB Privacy** | Attempt direct connection to RDS from local workshop. | Connection Refused (RDS in isolated subnet). | [ ] |
| **SEC-04** | **SSM Security** | Check `get-parameters.php` for SSM calls. | Credentials are encrypted and retrieved via IAM. | [ ] |
| **SEC-05** | **SG Layering** | View `RDS-SG` inbound rules in AWS Console. | Only `3306` from `EC2-SG` is permitted. | [ ] |

---

## 📈 Module 2: High Availability (HA)
**Objective:** Validate the system's resilience to component failure and AZ outages.

| ID | Test Category | Procedure | Expected Result | Status |
|:---:|---|---|---|:---:|
| **HA-01** | **Fault Tolerance** | Terminate instance `10.0.3.177` manually. | Site stays online; ASG launches new node automatically. | [ ] |
| **HA-02** | **ALB Balancing** | Successive refreshes of the Home Page. | Request cycles between `.177` and `.18`. | [ ] |
| **HA-03** | **DB Failover** | Trigger **Reboot with Failover** on RDS instance. | Standby promoted; Application reconnects in <60s. | [ ] |
| **HA-04** | **Health Checks** | Manually stop Apache on one instance. | ALB marks target `Unhealthy` and stops traffic.| [ ] |

---

## 🚀 Module 3: Scalability & Performance
**Objective:** Verify the Auto Scaling Group (ASG) responds to CPU demand dynamically.

| ID | Test Category | Procedure | Expected Result | Status |
|:---:|---|---|---|:---:|
| **SCA-01** | **Scale Out** | Run `stress` tool on EC2 to push CPU > 70%. | ASG adds instances (Max=4) to handle load. | [ ] |
| **SCA-02** | **Scale In** | Stop `stress` and wait for cooldown period. | ASG terminates extra instances; returns to Min. | [ ] |
| **SCA-03** | **Load Dist.** | Refresh site while Scaling is active. | Traffic is distributed across all 3-4 active IPs. | [ ] |

---

## 🔗 Module 4: Application & Data Integrity
**Objective:** Ensure application features work seamlessly across the distributed backend.

| ID | Test Category | Procedure | Expected Result | Status |
|:---:|---|---|---|:---:|
| **APP-01** | **RDS Queries** | Click "Query" and fetch Country Data. | Table displays data accurately from RDS database. | [ ] |
| **APP-02** | **Param Sync** | Verify `get-parameters.php` loads for all nodes. | All nodes share same config via SSM. | [ ] |
| **APP-03** | **Nav Integrity** | Click all links (GDP, Population, Mobile). | All data pages render without 404/500 errors. | [ ] |

---

## ⌨️ Appendix: Execution Commands

### 1. CPU Stress Simulation
Use these commands on an instance to trigger Auto Scaling:
```bash
# Update packages and install stress
sudo yum update -y
sudo yum install stress -y

# Stress test (2 workers for 5 minutes)
# Watch CloudWatch Alarms while this runs!
stress --cpu 2 --timeout 300s
```

### 2. Manual Health Check Failure
Simulate a web server crash to test ALB recovery:
```bash
# Stop Apache
sudo systemctl stop httpd

# Monitor Target Group health status in AWS Console
```

---
**Verified by:** ____________________  **Date:** 2026-04-20

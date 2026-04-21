# A3-G7 AWS Infrastructure — Testing & Validation Manual
**Project:** CT097-3-3-CSVC Cloud Infrastructure and Services
**Environment:** AWS us-east-1 (Production-Ready Demo)
**ALB URL:** http://a3-g7-csvc-alb-51735129.us-east-1.elb.amazonaws.com/
**Tested On:** 2026-04-21

---

## 📑 Table of Contents
1. [Pre-Test State Verification](#1-pre-test-state-verification)
2. [Module 1 — Security Validation](#module-1--security-validation)
3. [Module 2 — High Availability (HA) Failover Test](#module-2--high-availability-ha-failover-test)
4. [Module 3 — CPU Stress & Auto Scaling Test ⬅️ MAIN TEST](#module-3--cpu-stress--auto-scaling-test)
5. [Module 4 — Application & Data Integrity](#module-4--application--data-integrity)
6. [Expected Results Summary](#expected-results-summary)
7. [Troubleshooting](#troubleshooting)

---

## 1. Pre-Test State Verification

Before starting any test, confirm the current baseline state. All checks must pass before proceeding.

### 1.1 — Confirm 2 Instances Are Running

1. Open AWS Console → **EC2** → **Instances**
2. Filter: `Instance state = running`

Verify this exact table:

| Instance Name         | Private IP   | AZ          | State   | Status Checks     |
|-----------------------|--------------|-------------|---------|-------------------|
| A3-G7-CSVC-EC2 (ASG)  | 10.0.3.165   | us-east-1a  | Running | 2/2 checks passed |
| A3-G7-CSVC-EC2 (ASG)  | 10.0.4.120   | us-east-1b  | Running | 2/2 checks passed |

> ✅ **Expected:** Both instances running, 2 instances total.

---

### 1.2 — Confirm Target Group Has 2 Healthy Targets

1. EC2 → **Target Groups** → **A3-G7-CSVC-TG**
2. Click the **Targets** tab

| Target IP  | Port | AZ         | Health Status |
|------------|------|------------|---------------|
| 10.0.3.165 | 80   | us-east-1a | healthy       |
| 10.0.4.120 | 80   | us-east-1b | healthy       |

> ✅ **Expected:** Both targets show `healthy`.

---

### 1.3 — Confirm ASG Current State

1. EC2 → **Auto Scaling Groups** → **A3-G7-CSVC-ASG**
2. **Details** tab → verify capacity settings:

| Setting           | Value |
|-------------------|-------|
| Minimum capacity  | 1     |
| Desired capacity  | 2     |
| Maximum capacity  | 4     |

3. Click **Automatic scaling** tab → verify:
   - Policy name: `A3-G7-CSVC-CPU-Policy`
   - Type: `Target tracking scaling`
   - Metric: `Average CPU Utilization`
   - Target: `60%`

> ✅ **Expected:** Desired = 2, scaling policy set at 60% CPU.

---

### 1.4 — Confirm Website Is Live via ALB

Open any browser and go to:

```
http://a3-g7-csvc-alb-51735129.us-east-1.elb.amazonaws.com/index.php
```

> ✅ **Expected:** PHP page loads, country data visible, no errors.

---

## Module 1 — Security Validation

**Objective:** Confirm isolation of Three-Tier architecture.

### SEC-01 — EC2 Has No Public IP

1. EC2 → Instances → click on the instance with IP `10.0.3.165`
2. In the **Details** tab, check the **Public IPv4 address** field.

> ✅ **Expected:** Field is blank (dash). No public IP assigned. Direct internet access is impossible.

---

### SEC-02 — ALB Is the Only Entry Point

1. In your browser, try to directly access the private IP (this will fail from outside AWS):
   ```
   http://10.0.3.165/index.php
   ```
2. Then access via ALB:
   ```
   http://a3-g7-csvc-alb-51735129.us-east-1.elb.amazonaws.com/index.php
   ```

> ✅ **Expected:** Direct IP fails. ALB URL loads successfully.

---

### SEC-03 — RDS Security Group Allows Only Port 3306 from EC2-SG

1. EC2 → **Security Groups** → search for `A3-G7-CSVC-RDS-SG`
2. Click on it → **Inbound rules** tab

> ✅ **Expected:** Only 1 rule: `Type=MYSQL/Aurora, Port=3306, Source=A3-G7-CSVC-EC2-SG`.
> No 0.0.0.0/0 allowed on any port.

---

### SEC-04 — SSM Parameter Store (Credentials Not Hardcoded)

1. Open browser:
   ```
   http://a3-g7-csvc-alb-51735129.us-east-1.elb.amazonaws.com/get-parameters.php
   ```
2. Page should display parameters fetched live from SSM.

> ✅ **Expected:** Page loads and shows parameter values retrieved via IAM role, not hardcoded.

---

## Module 2 — High Availability (HA) Failover Test

**Objective:** Prove the system survives an instance failure with zero downtime.

> ⚠️ **Warning:** Do this test BEFORE the CPU stress test. After this test, ASG will automatically
> replace the stopped instance and restore 2 healthy instances before you continue.

---

### HA-01 — Manual Instance Failover

**Step 1 — Open the website in a browser tab and keep it open:**
```
http://a3-g7-csvc-alb-51735129.us-east-1.elb.amazonaws.com/index.php
```

**Step 2 — Terminate one instance:**
1. EC2 → Instances
2. Select the instance with IP **10.0.4.120**
3. Click **Instance state** → **Stop instance**
4. Confirm the stop

**Step 3 — Immediately keep refreshing the browser tab (F5 every 5 seconds)**

> ✅ **Expected:** Website continues to load on every refresh. No downtime.
> (ALB detects AZ2 instance is unhealthy and sends all traffic to AZ1 instance at `10.0.3.165`)

**Step 4 — Watch Target Group heal:**
1. EC2 → Target Groups → A3-G7-CSVC-TG → **Targets** tab
2. Refresh every 30 seconds

| Target IP  | Status During Failover |
|------------|------------------------|
| 10.0.3.165 | healthy                |
| 10.0.4.120 | unhealthy (then gone)  |

**Step 5 — Watch ASG auto-replace the instance:**
1. EC2 → Auto Scaling Groups → A3-G7-CSVC-ASG → **Activity** tab
2. After ~3-5 minutes you will see an entry like:
   ```
   Terminating EC2 instance: i-0cf5ba5d454cfcbd5
   Reason: Instance was taken out of service in response to a health check.

   Launching a new EC2 instance: i-XXXXXXXXXX
   Status: Successful
   ```

**Step 6 — Wait for recovery (approximately 5-10 minutes total):**
- New instance appears in EC2 Instances list with state `Running`
- Target Group shows 2 healthy targets again
- ASG Desired capacity = 2, Current = 2

> ✅ **Expected outcome:** System self-healed with zero manual action. 2 healthy instances restored automatically.

---

### HA-02 — ALB Load Distribution Verification

Once both instances are healthy again, refresh the ALB URL multiple times rapidly:
```
http://a3-g7-csvc-alb-51735129.us-east-1.elb.amazonaws.com/index.php
```

> ✅ **Expected:** ALB distributes requests across both AZs (round-robin).

---

## Module 3 — CPU Stress & Auto Scaling Test

**Objective:** Push CPU above 60% on BOTH instances simultaneously so the ASG `A3-G7-CSVC-CPU-Policy`
triggers a scale-out event and launches additional EC2 instances.

> ⚠️ **IMPORTANT — Why stress BOTH instances:**
> The scaling policy measures **average CPU across ALL instances in the ASG**.
> If only 1 of 2 instances is stressed → average = (100% + ~2%) / 2 ≈ **51%** → will NOT trigger scale-out.
> If BOTH instances are stressed → average = (100% + 100%) / 2 = **100%** → will DEFINITELY trigger scale-out.
>
> You need **two browser tabs open simultaneously** — one Session Manager session per instance.

---

### Step 1 — Open Session Manager for Instance AZ1 (10.0.3.165)

1. AWS Console → **Systems Manager** (search "Systems Manager" in the top search bar)
2. Left sidebar → **Session Manager**
3. Click **Start session** (orange button)
4. In the list, find the instance with Private IP `10.0.3.165`
5. Select it → Click **Start session**
6. A terminal window opens in the browser — **keep this tab open**

---

### Step 2 — Open Session Manager for Instance AZ2 (10.0.4.120)

1. Open a **new browser tab**
2. AWS Console → **Systems Manager** → **Session Manager**
3. Click **Start session**
4. Find the instance with Private IP `10.0.4.120`
5. Select it → Click **Start session**
6. A second terminal window opens — **keep this tab open**

You now have **two terminal tabs** — one for each EC2 instance.

---

### Step 3 — Install the `stress` Tool on BOTH Instances

Run this command in **Tab 1 (AZ1 - 10.0.3.165)**:
```bash
sudo yum install -y stress
```

Run this command in **Tab 2 (AZ2 - 10.0.4.120)**:
```bash
sudo yum install -y stress
```

Wait for both to complete. You will see:
```
Installed:
  stress-1.0.4-28.amzn2023.x86_64
Complete!
```

> ✅ If `stress` is already installed, you will see: `Package stress-1.0.4 is already installed.`

---

### Step 4 — Open CloudWatch Monitoring (in a third browser tab)

Before starting the stress test, open monitoring so you can watch CPU in real time:

1. Open a **new browser tab**
2. AWS Console → **EC2** → **Auto Scaling Groups** → **A3-G7-CSVC-ASG**
3. Click the **Monitoring** tab
4. Scroll to **EC2 metrics** section
5. Find **CPU Utilization** graph

> Keep this tab open throughout the test. Refresh it every minute.

---

### Step 5 — Launch Stress Test on BOTH Instances Simultaneously

**In Tab 1 (AZ1 — 10.0.3.165)**, run:
```bash
stress --cpu 2 --timeout 300
```

**IMMEDIATELY switch to Tab 2 (AZ2 — 10.0.4.120)** and run:
```bash
stress --cpu 2 --timeout 300
```

> **What this does:**
> - `--cpu 2` → spawns 2 workers (matches t2.small's 2 vCPUs) → drives each instance to ~100% CPU
> - `--timeout 300` → runs for exactly 5 minutes (300 seconds) then stops automatically
> - Both instances at ~100% CPU → average = 100% → far exceeds the 60% threshold

The terminal will show:
```
stress: info: [XXXX] dispatching hogs: 2 cpu, 0 io, 0 vm, 0 hdd
```
This means it is running. The terminal will appear to hang — **this is normal**, it is working.

---

### Step 6 — Monitor Scale-Out in Real Time

While the stress test runs (you have 5 minutes), watch these **in the third browser tab**:

#### 6A — Watch ASG Activity Tab (most important)
1. EC2 → Auto Scaling Groups → A3-G7-CSVC-ASG → **Activity** tab
2. Refresh every 60 seconds
3. After **1-2 minutes** you will see a new entry appear:
   ```
   Launching a new EC2 instance: i-XXXXXXXXXXXXXXXXX
   Reason: Monitor "A3-G7-CSVC-CPU-Policy" triggered a scale-out activity.
   Status: Successful
   ```

> ✅ **This is the proof that Auto Scaling worked.**

#### 6B — Watch CPU Utilization in Monitoring Tab
1. EC2 → Auto Scaling Groups → A3-G7-CSVC-ASG → **Monitoring** tab
2. Under EC2 metrics → **CPU Utilization** graph
3. You will see the line spike sharply from ~2% to ~90-100%

#### 6C — Watch New Instance Launch in EC2 Instances
1. EC2 → **Instances**
2. Refresh every 30 seconds
3. A 3rd instance will appear (named `A3-G7-CSVC-EC2-AZ2`) with state `Pending` then `Running`
4. Instance count will go from **2 → 3** (possibly up to 4 if load is sustained)

#### 6D — Watch Target Group Register New Instance
1. EC2 → Target Groups → A3-G7-CSVC-TG → **Targets** tab
2. Refresh every 30 seconds
3. New target IP will appear with status `initial` → then `healthy`

---

### Step 7 — Take Screenshots (Required for Submission)

While the scale-out is happening, take these screenshots:

| # | What to Screenshot | Where to Find It |
|---|---|---|
| 1 | CPU spike graph | ASG → Monitoring → CPU Utilization |
| 2 | Scale-out activity log | ASG → Activity tab → entry showing "triggered a scale-out activity" |
| 3 | 3 instances running | EC2 → Instances (show 3 running instances) |
| 4 | Target Group with 3 healthy targets | EC2 → Target Groups → A3-G7-CSVC-TG → Targets |
| 5 | Website still working during load | Browser showing ALB URL loaded |

---

### Step 8 — Stop the Stress Test (Scale-In)

The stress tool will auto-stop after 300 seconds. If you want to stop it manually:

**In Tab 1 (AZ1):** Press `Ctrl + C`

**In Tab 2 (AZ2):** Press `Ctrl + C`

After stopping:
- CPU drops back to ~2% on both instances
- The `A3-G7-CSVC-CPU-Policy` will detect average CPU has dropped below 60%
- ASG begins **scale-in** after a cooldown period (~15 minutes)

---

### Step 9 — Verify Scale-In (Return to 2 Instances)

Wait approximately **15 minutes** after stopping the stress test. Then:

1. EC2 → Auto Scaling Groups → A3-G7-CSVC-ASG → **Activity** tab
2. You will see:
   ```
   Terminating EC2 instance: i-XXXXXXXXXXXXXXXXX
   Reason: Scale in
   Status: Successful
   ```
3. EC2 → Instances: count returns to **2 instances**
4. ASG → Details: Desired capacity = 2, Current = 2

> ✅ **Expected:** ASG automatically removes the extra instance. Returns to baseline of 2.

---

### Scale Test — Full Expected Timeline

| Time        | Event |
|-------------|-------|
| T+0:00      | Stress test started on both instances |
| T+0:30      | CPU visible rising in CloudWatch |
| T+1:00–2:00 | ASG detects average CPU > 60% |
| T+1:30–2:30 | ASG Activity shows "Launching a new EC2 instance" |
| T+2:00–3:00 | 3rd instance appears in EC2 Instances (Pending) |
| T+3:00–5:00 | 3rd instance reaches Running state |
| T+5:00      | Stress tool auto-stops (timeout=300s) |
| T+5:30      | CPU drops back below 60% |
| T+20:00     | ASG terminates extra instance (scale-in after cooldown) |
| T+21:00     | Back to 2 instances, Desired=2 |

---

## Module 4 — Application & Data Integrity

**Objective:** Ensure all application pages work correctly and data is consistent.

### APP-01 — All Navigation Links Work

Open each URL in browser and verify no 404 or 500 error:

| Page | URL | Expected Result |
|---|---|---|
| Home / Index | `http://a3-g7-csvc-alb-51735129.us-east-1.elb.amazonaws.com/index.php` | Page loads, shows country data |
| GDP Data | `http://a3-g7-csvc-alb-51735129.us-east-1.elb.amazonaws.com/gdp.php` | GDP chart/table loads |
| Population | `http://a3-g7-csvc-alb-51735129.us-east-1.elb.amazonaws.com/population.php` | Population data loads |
| Life Expectancy | `http://a3-g7-csvc-alb-51735129.us-east-1.elb.amazonaws.com/lifeexpectancy.php` | Life expectancy data loads |
| Mortality | `http://a3-g7-csvc-alb-51735129.us-east-1.elb.amazonaws.com/mortality.php` | Mortality data loads |
| Mobile | `http://a3-g7-csvc-alb-51735129.us-east-1.elb.amazonaws.com/mobile.php` | Mobile data loads |
| Search Interface | `http://a3-g7-csvc-alb-51735129.us-east-1.elb.amazonaws.com/query.php` | Query selection form loads |
| Query Results | `http://a3-g7-csvc-alb-51735129.us-east-1.elb.amazonaws.com/query2.php` | Filtered RDS data displays correctly |

---

### APP-02 — Database Query Validation

1. Connect to AZ1 instance via Session Manager (see Module 3 Steps 1–2)
2. Run:
```bash
mysql -h $(aws ssm get-parameter --name /example/endpoint --query Parameter.Value --output text) \
      -u $(aws ssm get-parameter --name /example/username --query Parameter.Value --output text) \
      -p$(aws ssm get-parameter --name /example/password --query Parameter.Value --with-decryption --output text) \
      countries
```
3. Once connected, run:
```sql
SELECT COUNT(*) FROM countrydata_final;
```

> ✅ **Expected:** Returns `215` (total countries in database).

```sql
SELECT name, population, GDP FROM countrydata_final WHERE name = 'United States';
```

> ✅ **Expected:** Returns a valid row with US population and GDP values.

```sql
exit
```

---

### APP-03 — Verify SSM Parameter Store Access

In the Session Manager terminal on either instance, run:
```bash
aws ssm get-parameter --name /example/endpoint --region us-east-1
aws ssm get-parameter --name /example/username --region us-east-1
aws ssm get-parameter --name /example/database --region us-east-1
```

> ✅ **Expected:** Each command returns a JSON response with `"Value": "..."`. No `AccessDeniedException`.

---

## Expected Results Summary

| Test ID | Test Name | Expected Result | Status |
|:---:|---|---|:---:|
| SEC-01 | EC2 has no public IP | No public IP in instance details | [ ] |
| SEC-02 | ALB is sole entry point | Direct IP fails, ALB URL works | [ ] |
| SEC-03 | RDS-SG allows only 3306 from EC2-SG | Confirmed in console | [ ] |
| SEC-04 | SSM credentials not hardcoded | get-parameters.php loads via IAM | [ ] |
| HA-01 | Instance failover — no downtime | Site stays up, ASG replaces instance | [ ] |
| HA-02 | ALB distributes traffic across AZs | Requests hit both AZs | [ ] |
| SCA-01 | CPU stress triggers scale-out | 3rd instance launches when CPU > 60% | [ ] |
| SCA-02 | Scale-in after stress ends | Returns to 2 instances after cooldown | [ ] |
| SCA-03 | New instance takes live traffic | Target Group shows 3 healthy targets | [ ] |
| APP-01 | All pages load without errors | All 6 URLs return HTTP 200 | [ ] |
| APP-02 | Database has 215 rows | `COUNT(*)` returns 215 | [ ] |
| APP-03 | SSM parameters accessible | `aws ssm get-parameter` returns values | [ ] |

---

## Troubleshooting

| Problem | Likely Cause | Fix |
|---|---|---|
| Scale-out not triggered after 5 min | Only one instance was stressed (average < 60%) | Re-run stress on BOTH instances simultaneously |
| Session Manager shows no instances | SSM Agent not running or IAM role missing | Verify `Inventory-App-Role` is attached to EC2 |
| `stress: command not found` | Package not installed | Run `sudo yum install -y stress` again |
| CPU shows high in CloudWatch but no new instance | Alarm cooldown period still active | Wait 2 more minutes and refresh Activity tab |
| New instance stays `unhealthy` in Target Group | Instance warmup period (300s) | Wait 5 minutes after instance reaches Running |
| Website returns 502 during scale test | Normal briefly as new instances warm up | Refresh after 30 seconds; ALB handles it |
| `mysql` command not found in Session Manager | MySQL client not installed | Use `sudo yum install -y mysql` first |

---

**Verified by:** ____________________  **Date:** 2026-04-21
**ALB Endpoint:** http://a3-g7-csvc-alb-51735129.us-east-1.elb.amazonaws.com/
**AZ1 Instance:** 10.0.3.165 (A3-G7-CSVC-EC2 (ASG), us-east-1a)
**AZ2 Instance:** 10.0.4.235 (A3-G7-CSVC-EC2 (ASG), us-east-1b)

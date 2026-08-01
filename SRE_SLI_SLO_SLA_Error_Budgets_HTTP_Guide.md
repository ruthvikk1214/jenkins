# Master SRE Guide: Complete AutoRABIT Preparation Package

This ultimate master guide covers **100% of the requirements from the AutoRABIT Site Reliability & Cloud Platform Engineer Job Description**:

- **SLIs/SLOs/SLAs & Enforcing Error Budgets**
- **Observability & ELK Stack Engineering** (Logs, Metrics, Alerts)
- **Incident Response, On-Call, RCA & Blameless Postmortems**
- **Automating Infrastructure using Terraform**
- **CIS Benchmarks Level 1/2, SOC 2 Type II & ISO 27001 Security Controls**
- **Database Operations & Performance Tuning** (PostgreSQL, MySQL, DynamoDB)
- **Multi-Region High Availability & Disaster Recovery (DR) Architecture**
- **AWS Compute Operations** (EKS, ECS, EC2, Lambda, SSM Session Manager)
- **AIOps & Amazon Bedrock** (Log Anomaly Detection & Postmortem Summaries)
- **Kyndryl Linux Sysadmin $\rightarrow$ SRE Transition Strategy & Elevator Pitch**

---

# SECTION 1: SLIs, SLOs, SLAs, Error Budgets & HTTP Status Code Engineering

## 1. The Core Hierarchy: SLI ──► SLO ──► SLA

```
                  ┌────────────────────────┐
                  │          SLA           │  <-- Business / Legal Agreement (Financial Penalties)
                  ├────────────────────────┤
                  │          SLO           │  <-- Internal Target (Goal set by SRE + Product)
                  ├────────────────────────┤
                  │          SLI           │  <-- Real-Time Metric (Measured via HTTP status codes & latency)
                  └────────────────────────┘
```

| Term | Full Name | Definition | Who Cares? | Real-World Example |
| :--- | :--- | :--- | :--- | :--- |
| **SLI** | Service Level Indicator | Quantitative measure of real-time service performance. | SREs, Developers, Observability tools | *"99.94% of API requests returned HTTP 2xx/3xx in < 200ms over 30 days."* |
| **SLO** | Service Level Objective | Target agreed upon by SRE & Product teams. | SREs, Product Managers, Tech Leads | *"The service must maintain 99.9% success rate over a 30-day rolling window."* |
| **SLA** | Service Level Agreement | Contract specifying financial/legal penalties if SLO fails. | Customers, Legal, Executives, Sales | *"If monthly availability falls below 99.5%, customers get a 15% bill credit."* |

> 💡 **Golden Rule**: **$SLO > SLA$**. Your internal target ($SLO = 99.9\%$) must **always be higher** than your external contract ($SLA = 99.5\%$) so you have a safety cushion to fix issues before paying penalties.

---

## 2. HTTP Status Codes & SLI Classification

```
HTTP Requests Ingestion
       │
       ├──► 2xx / 3xx ─────────────────► [GOOD EVENT] ──► Counts towards SLI Success
       │
       ├──► 5xx (500, 502, 503, 504) ──► [BAD EVENT]  ──► BURNS ERROR BUDGET!
       │
       └──► 4xx (400, 401, 403, 404) ──► [EXCLUDED]   ──► Client Fault (Does NOT burn budget)
```

### Detailed HTTP Status Code Breakdown Table

| HTTP Status Code | Meaning | SLI Category | Impact on Error Budget | SRE / System Root Cause |
| :--- | :--- | :--- | :--- | :--- |
| **`200 OK` / `201 Created` / `204 No Content`** | Request Succeeded | **GOOD** | Increases SLI | Normal healthy application response. |
| **`301` / `302` / `304 Not Modified`** | Redirect / Cached | **GOOD** | Increases SLI | Normal redirection or browser cache hit. |
| **`400 Bad Request`** | Malformed Payload | **EXCLUDED** | **NO BURN** | Client sent invalid JSON/parameters. |
| **`401 Unauthorized` / `403 Forbidden`** | Auth Failure | **EXCLUDED** | **NO BURN** | Invalid API token or insufficient RBAC. |
| **`404 Not Found`** | Resource Missing | **EXCLUDED** | **NO BURN** | Client called wrong URL endpoint. |
| **`429 Too Many Requests`** | Rate Limited | **EXCLUDED / BAD** | **CONDITIONAL** | Excluded if client exceeds quota; counted as Bad if gateway throttles valid users. |
| **`500 Internal Server Error`** | App Crash / Unhandled | **BAD** | 🔥 **BURNS BUDGET** | Code exception, unhandled NullPointer, DB fail. |
| **`502 Bad Gateway`** | Upstream Unreachable | **BAD** | 🔥 **BURNS BUDGET** | Pod crashed (`CrashLoopBackOff`), Nginx/ALB cannot connect to backend. |
| **`503 Service Unavailable`** | Server Overloaded | **BAD** | 🔥 **BURNS BUDGET** | Pod CPU/RAM saturated, DB connection pool exhausted, queue full. |
| **`504 Gateway Timeout`** | Upstream Timeout | **BAD** | 🔥 **BURNS BUDGET** | Query execution timed out (>30s), downstream microservice hanging. |

---

## 3. Mathematical SLI Formulas with HTTP Status Codes

### A. Availability SLI Formula (Request-Based)

$$\text{Availability SLI} = \left( \frac{\text{Count of HTTP }(2xx + 3xx)}{\text{Total HTTP Requests} - \text{Count of HTTP }(4xx)} \right) \times 100$$

#### PromQL (Prometheus) Implementation:
```promql
# PromQL for 30-day Rolling HTTP Availability SLI
(
  sum(rate(http_requests_total{status=~"2..|3.."}[30d]))
  /
  sum(rate(http_requests_total{status=!~"4.."}[30d]))
) * 100
```

---

## 4. Error Budget Mechanics & Downtime Table

### Error Budget Formula:
$$\text{Error Budget} = 100\% - \text{SLO}$$

#### Allowed Downtime Reference Table (Monthly - 30 Days / 720 Hours):

| SLO Target | Error Budget (%) | Allowed Downtime per Month | Allowed Downtime per Year | Max 5xx Errors (per 10M Requests) |
| :--- | :--- | :--- | :--- | :--- |
| **99%** ("Two Nines") | `1.0%` | **7 hours, 12 minutes** | 3.65 days | 100,000 |
| **99.5%** | `0.5%` | **3 hours, 36 minutes** | 1.83 days | 50,000 |
| **99.9%** ("Three Nines") | `0.1%` | **43 minutes, 12 seconds** | 8.76 hours | 10,000 |
| **99.99%** ("Four Nines") | `0.01%` | **4 minutes, 19 seconds** | 52.56 minutes | 1,000 |

---

## 5. Burn Rate & Error Budget Enforcement Policy

$$\text{Burn Rate} = \frac{\text{Actual HTTP 5xx Rate}}{100\% - \text{SLO}}$$

- **Fast Burn Alert (P1 Urgent)**: Alert if **2% of monthly budget is burned in 1 hour** (Burn rate = 14.4). Page On-Call SRE via PagerDuty.
- **Slow Burn Alert (P2 Warning)**: Alert if **5% of monthly budget is burned in 6 hours** (Burn rate = 6). Notify team via Slack channel.

### What happens when Error Budget hits 0%?
1. **CI/CD Pipeline Lock**: GitHub Actions / CodePipeline blocks non-hotfix feature deployment jobs automatically.
2. **Reliability Sprint Pivot**: Developers stop new feature work and spend 100% of sprint capacity fixing root causes (e.g. adding PgBouncer for 503 DB errors, setting up Karpenter for 502 OOMKilled crashes).
3. **Unfreezing Rule**: Feature deployments resume ONLY when 30-day rolling budget restores above **20%**.

---

# SECTION 2: Build & Maintain Observability using Elasticsearch & Kibana (ELK)

## 1. What is ELK & The ELK Stack?

**ELK** stands for:
- **E** = **E**lasticsearch (The Search & Storage Engine)
- **L** = **L**ogstash (The Data Processor & Transformer)
- **K** = **K**ibana (The Visualization & Alerting UI)

Together with **Beats** (Filebeat/Metricbeat), it is officially known as the **Elastic Stack**.

---

## 2. What is Observability & The 3 Pillars (MEL Triad)

- **Monitoring** tells you: **"WHAT is broken?"** *(e.g., Server CPU is at 99%)*
- **Observability** allows you to ask: **"WHY is it broken?"** *(e.g., Why CPU is 99%? User 123 ran an un-indexed SQL query on pod-release-4)*

### The 3 Pillars of Observability:
1. **Logs**: Detailed text event records (Filebeat + Logstash + Elasticsearch).
2. **Metrics**: Countable numeric health stats over time (Metricbeat + Elasticsearch).
3. **Traces**: The path of a request hopping across microservices (Elastic APM + Kibana).

---

## 3. Simple Real-World Analogy: The Giant Library

1. **Filebeat / FluentBit (The Mailmen 📬)**: Stands on every server, collects paper notes (logs) the second they are written.
2. **Logstash (The Editors ✍️)**: Cleans up messy handwriting (raw text) into neat index cards (`Time=2:00PM | Level=ERROR | Pod=release-1`). This is **Grok Parsing**.
3. **Elasticsearch (The Superfast Filing Warehouse 🗄️)**: Stores millions of index cards using smart **Shards** and **Lucene Inverted Indexes** to return search results in milliseconds.
4. **Kibana (The Screen & Alarm Bell 🚨)**: Renders interactive charts and rings a loud alarm (PagerDuty page) if 50 error cards arrive in 1 minute.

---

## 4. Technical Functionality Breakdown

| Tool | Technical Category | Core Technical Functionality |
| :--- | :--- | :--- |
| **Filebeat / FluentBit** | Lightweight Agent / Shipper | Tails files on disk, handles offset tracking for restart recovery, manages backpressure, combines multiline Java stack traces, ships over TLS. |
| **Logstash** | ETL Processing Engine | **Input**: Receives Beats/Kafka. **Filter**: Executes Grok RegEx parsing, Mutate type-casting, Date ISO-8601 formatting, GeoIP resolution. **Output**: Routes JSON to Elasticsearch. |
| **Elasticsearch** | Search Engine & NoSQL Database | Uses Apache Lucene **Inverted Indexing** (mapping terms $\rightarrow$ Doc IDs) for sub-second searches. Handles primary/replica sharding and compute aggregations. |
| **Kibana** | Visualization & Alert Engine | Translates UI/KQL into Elasticsearch JSON DSL queries, renders dashboards, executes threshold alerting checks to PagerDuty/Slack. |

---

## 5. The 3 Memory & Shard Rules Every SRE Must Know

1. **30GB JVM Heap Rule**: Never allocate more than **30GB-32GB** to JVM Heap. Above ~32GB, Java disables **Compressed Object Pointers (Compressed OOPs)**, dropping performance by 30%.
2. **50/50 RAM Split**: Assign **50% RAM to JVM Heap** (max 30GB), leave **50% for Linux OS Page Cache** (Lucene disk search).
3. **Shard Sizing Rule**: Keep target shard size between **20GB and 50GB**. Keep total shards per node **$< 20 \text{ shards per GB Heap}$.**

---

# SECTION 3: Incident Response, On-Call, RCA & Blameless Postmortems

## 1. The Incident Response Lifecycle

```
[1. Detection & Alerting] ──► [2. Triage & Severity] ──► [3. Incident Commander Assigned]
                                                                  │
[7. Preventative Action Items] ◄── [6. RCA & Postmortem] ◄── [5. Restoration] ◄── [4. Containment / Mitigation]
```

### Phase-by-Phase Execution:

| Phase | Core Objective | Key SRE Tools / Actions |
| :--- | :--- | :--- |
| **1. Detection** | Detect anomaly before customers report it. | Kibana Alerts, CloudWatch Alarms, PagerDuty, Prometheus Burn Rate. |
| **2. Triage** | Determine severity (P1 vs P2) and business impact. | Define scope (e.g. 30% of enterprise tenants affected). |
| **3. Command** | Establish clear command structure. | Declare Incident Commander (IC), open War Room (Slack / Zoom). |
| **4. Mitigation** | **Stop the bleeding!** (DO NOT diagnose root cause yet). | Roll back release (`helm rollback`), shed non-critical traffic, scale nodes. |
| **5. Restoration** | Verify system stability and clear alarms. | Monitor 4 Golden Signals (Latency, Traffic, Errors, Saturation) for 30 mins. |
| **6. Postmortem** | Conduct 5-Whys RCA and document timeline. | Blameless Postmortem meeting within 48-72 hours. |
| **7. Remediation** | Implement preventative fixes. | Create Jira tickets with strict SLA completion deadlines. |

---

## 2. Incident Command System (ICS) Roles

1. **Incident Commander (IC)**: Single point of authority. Controls the Incident War Room, delegates tasks, approves service degradation trade-offs. (IC does NOT type CLI commands).
2. **Operations Lead (OL)**: Technical lead executing mitigation commands, traffic rerouting, or rollbacks.
3. **Communications Lead (CL)**: Sends internal updates every 15–30 minutes (Slack `#incidents-p1`) and updates public StatusPage.
4. **Subject Matter Experts (SMEs)**: Domain engineers (DBA, EKS Specialist) analyzing metrics/logs.

---

## 3. Root Cause Analysis (RCA): The 5-Whys Framework

> *"Human error is NEVER the root cause. Human error is the starting point of an investigation."*

### Real-World 5-Whys Example (AutoRABIT Production Outage):
* **Problem Statement**: AutoRABIT API returned `HTTP 503 Service Unavailable` for 25 minutes.
1. **Why?** $\rightarrow$ All PostgreSQL database connections were maxed out.
2. **Why were DB connections maxed out?** $\rightarrow$ A background metadata sync job opened 500 parallel queries without closing them.
3. **Why did the job open 500 unclosed connections?** $\rightarrow$ The application lacked connection pooling and query timeout parameters.
4. **Why was there no connection pool or query timeout?** $\rightarrow$ PgBouncer was not deployed in front of RDS, and default GORM timeouts were infinite.
5. **Why was PgBouncer not deployed?** $\rightarrow$ Database Terraform modules lacked mandatory connection pool configuration standards during initial provisioning.

**Systemic Root Cause**: Missing connection pooling architecture standards in Terraform modules.

---

# SECTION 4: Automating Infrastructure using Terraform

## 1. Enterprise Terraform Architecture & State Management

### A. Remote Backend Configuration with State Locking

```hcl
# backend.tf
terraform {
  required_version = ">= 1.5.0"

  backend "s3" {
    bucket         = "autorabit-terraform-state-prod"
    key            = "eks/us-east-1/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "autorabit-tf-locks"
    encrypt        = true
  }

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}
```

---

## 2. Refactoring & Drift Detection

### A. The `moved {}` Block (Terraform 1.1+)
```hcl
moved {
  from = aws_security_group.web_sg
  to   = module.vpc.aws_security_group.web_sg[0]
}
```

### B. Detecting Drift via CLI Exit Codes:
```bash
terraform plan -detailed-exitcode
```
- **Exit Code `0`**: Match.
- **Exit Code `2`**: **Drift Detected!**

---

# SECTION 5: CIS Benchmarks, SOC 2 Type II & ISO 27001 Security Controls

## 1. CIS Level 1 vs Level 2 Linux OS Hardening

- **CIS Level 1**: Essential, non-disruptive baseline.
- **CIS Level 2 (Defense-in-Depth)**: Strict kernel parameters, mount flags, and audit rules for high-security enterprise environments.

### A. Kernel Hardening Parameters (`/etc/sysctl.d/99-cis-hardening.conf`):
```ini
net.ipv4.ip_forward = 0
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.default.accept_redirects = 0
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.default.rp_filter = 1
fs.suid_dumpable = 0
```

### B. Partition Hardening Flags (`/etc/fstab`):
```fstab
tmpfs   /tmp        tmpfs   defaults,rw,nosuid,nodev,noexec   0 0
tmpfs   /var/tmp    tmpfs   defaults,rw,nosuid,nodev,noexec   0 0
tmpfs   /dev/shm    tmpfs   defaults,rw,nosuid,nodev,noexec   0 0
```

---

## 2. SOC 2 Type II & ISO 27001 Mapping

- **SOC 2 CC6.1 (Access Control)**: AWS IAM least privilege, STS temporary tokens, and **AWS SSM Session Manager** (no Port 22 SSH).
- **SOC 2 CC6.8 (Malware & Threat Defense)**: **Trend Micro Deep Security Agents (DSA)** for File Integrity Monitoring (FIM) and Intrusion Prevention (IPS).
- **ISO 27001 A.8.12 (Data Leakage)**: AWS KMS encryption at rest (EBS, S3, RDS) & TLS 1.3 in transit.

---

# SECTION 6: Database Operations, Performance Tuning & Multi-Region HA

## 1. Relational Databases: PostgreSQL & MySQL (RDS / Aurora)

### A. PostgreSQL SRE Performance Tuning Parameters
- **`shared_buffers`**: **25% of total node RAM** for caching table pages.
- **`work_mem`**: **64MB – 256MB** per query operation.
- **`effective_cache_size`**: **75% of total node RAM** planner hint.
- **PgBouncer**: Transaction-level connection pooler multiplexing 5,000 client connections over 50 persistent backend DB connections to prevent CPU saturation (`HTTP 503`).

### B. Troubleshooting Query Bottlenecks (`pg_stat_statements`)
```sql
SELECT query, calls, total_exec_time, mean_exec_time
FROM pg_stat_statements ORDER BY mean_exec_time DESC LIMIT 5;
```

## 2. NoSQL Databases: Amazon DynamoDB
- **Partition Key Hashing**: Avoid hot keys (like `Date`) by using composite high-cardinality keys (`PK = TenantID#OrgID#Timestamp`).
- **Amazon DAX**: In-memory write-through cache delivering microsecond read latency.

## 3. Multi-Region High Availability & Disaster Recovery (DR)

```
Cost & Complexity:  Low ──────────────────────────────────────────────────────────► High
Strategy:          [Backup & Restore] ──► [Pilot Light] ──► [Warm Standby] ──► [Active-Active]
RTO / RPO:         Hours                  Minutes           Seconds             Near-Zero / Zero
```

- **Amazon Aurora Global Database**: Storage-level cross-region replication with **`< 1 second` replication lag**.
- **DynamoDB Global Tables**: Active-active multi-region replication.
- **AWS Route 53**: Latency and automated health-check failover routing.

---

# SECTION 7: AWS Compute Operations (EKS, ECS, EC2, Lambda, SSM)

- **Amazon EKS**: Managed Kubernetes with **Karpenter** just-in-time node autoscaling, **IRSA** OIDC authorization, and **VPC CNI** IP prefix delegation (`ENABLE_PREFIX_DELEGATION=true`).
- **Amazon ECS**: Fargate serverless containers vs EC2 launch types. Task Definitions and Services.
- **Amazon EC2**: Auto Scaling Groups (ASGs) with Spot instances (saving 70-90%), `gp3` EBS volumes, Packer Golden AMIs.
- **AWS Lambda**: Serverless compute (< 15 mins execution limit), Reserved vs Provisioned Concurrency (cold start elimination).
- **AWS SSM**: **Session Manager** (bastion-less, SSH-less access with 0 open Port 22), Parameter Store, Patch Manager.

---

# SECTION 8: STAR Interview Model Answers

### Question: "How do you define SLIs using HTTP status codes, and what do you do when developers keep breaking the SLO?"
**Model Answer**:
> *"When defining Availability SLIs, I map HTTP response codes directly into Good, Bad, and Excluded events:
> - **Good Events**: HTTP 2xx and 3xx responses meeting latency thresholds (< 300ms).
> - **Bad Events**: HTTP 5xx errors—specifically `500` (code crashes), `502` (pod crashes/bad gateway), `503` (capacity overload), and `504` (database timeouts). These directly burn our Error Budget.
> - **Excluded Events**: HTTP 4xx client errors (`400`, `401`, `403`, `404`), as they represent client-side validation or auth failures, not platform unreliability.
>
> If developers are pushing releases that cause HTTP 5xx spikes and burn through the monthly Error Budget, our automated **Error Budget Policy** triggers a feature freeze in CodePipeline. The team pivots 100% of sprint capacity to reliability fixes—such as configuring PgBouncer for database connection timeouts or setting up Karpenter for EKS node autoscaling—until the rolling Error Budget recovers."*

---

### Question: "How do you handle a P1 outage from alert to postmortem?"
**Model Answer**:
> *"When a P1 outage occurs, my immediate focus is **mitigation—stopping the bleeding**—not deep root cause diagnosis.
> 1. **Response & Command**: Upon receiving a PagerDuty page, I acknowledge the alert, assume the role of Incident Commander (IC), and spin up a dedicated War Room. I delegate roles: an Operations Lead to run diagnostic CLI commands and a Communications Lead to update our StatusPage every 15 minutes.
> 2. **Mitigation**: We inspect our 4 Golden Signals. If a new deployment caused a spike in HTTP 502/503 errors, I authorize an immediate rollback via `helm rollback` or traffic shedding.
> 3. **RCA & Blameless Postmortem**: Within 48 hours, I facilitate a Blameless Postmortem with SREs and Dev leads using the **5-Whys methodology** to focus on systemic flaws (e.g. missing connection pooling) rather than human error.
> 4. **Preventative Action**: We generate concrete action items in Jira with clear completion SLAs (P0 items done in 3 days) to ensure technical debt is fixed permanently."*

---

### Question: "How do you optimize a slow PostgreSQL database supporting high-concurrency microservices?"
**Model Answer**:
> *"I optimize PostgreSQL through connection management, parameter tuning, and query analysis:
> 1. **Connection Pooling**: To stop connection churn, I deploy **PgBouncer** in transaction pooling mode. This allows thousands of client connections to multiplex over a stable pool of ~50 backend DB connections, preventing CPU exhaustion.
> 2. **Memory Tuning**: I set `shared_buffers` to 25% of node RAM for data caching, set `work_mem` between 64MB-256MB to avoid query disk spill, and set `effective_cache_size` to 75% RAM.
> 3. **Query Optimization**: Using `pg_stat_statements`, I identify slow queries and run `EXPLAIN (ANALYZE, BUFFERS)` to replace sequential scans with composite B-tree indexes."*

---

### Question: "How do you design a Multi-Region Active-Active Disaster Recovery architecture on AWS?"
**Model Answer**:
> *"For near-zero RTO/RPO multi-region availability:
> 1. **Data Layer**: We use **Amazon Aurora Global Database** for relational data (storage-level cross-region replication under 1 second) and **DynamoDB Global Tables** for active-active NoSQL replication.
> 2. **Compute & Ingress**: We deploy identical EKS clusters in both regions (`us-east-1` and `us-west-2`) managed by Terraform.
> 3. **Traffic Management**: **Route 53 Latency Routing** with automated DNS failover health checks routes users to the closest healthy region, ensuring zero disruption if an entire AWS region experiences an outage."*

---

# SECTION 11: Kubernetes Production Incident Response & Real-World Case Study

## 1. The Kubernetes Production Incident Framework

In a production environment, the golden rule of incident response is:  
👉 **Contain first (stop the bleeding), analyze second (find the root cause).**

```
Alert Received ➔ Triage & Scope ➔ Mitigate (Restore Service) ➔ Investigate RCA ➔ Deploy Fix ➔ Post-Mortem
```

---

### Step 1: Triage & Blast Radius Assessment
**Goal:** Determine what is broken, which namespace/service is affected, and whether the issue is cluster-wide or localized.

1. **Check Node & Cluster Health:**
   ```bash
   kubectl get nodes -o wide
   ```
   *Look for:* Nodes in `NotReady` state or experiencing `MemoryPressure`, `DiskPressure`, or `PIDPressure`.

2. **Identify Failing Pods Across Namespaces:**
   ```bash
   kubectl get pods -A --field-selector status.phase!=Running
   ```
   *Look for:* Pods in `CrashLoopBackOff`, `OOMKilled`, `ImagePullBackOff`, `Pending`, or `Terminating`.

3. **Identify Pods with High Restart Counts:**
   ```bash
   kubectl get pods -n <namespace> --sort-by='.status.containerStatuses[0].restartCount'
   ```

---

### Step 2: Containment & Immediate Mitigation
**Goal:** Restore availability for users immediately before diving into hours of debugging.

* **Scenario A: Issue started immediately after a deployment**  
  Roll back to the previous stable revision:
  ```bash
  kubectl rollout undo deployment/<deployment-name> -n <namespace>
  ```

* **Scenario B: Issue is caused by a sudden traffic spike**  
  Scale up replicas to share load:
  ```bash
  kubectl scale deployment/<deployment-name> --replicas=10 -n <namespace>
  ```

* **Scenario C: Specific pod is flapping or stuck**  
  Remove the pod from the service load balancer selector by changing its labels so you can inspect it in isolation without routing customer traffic to it:
  ```bash
  kubectl label pod <pod-name> -n <namespace> app=isolated-debug --overwrite
  ```

---

### Step 3: Deep-Dive Root Cause Analysis (RCA)
**Goal:** Gather empirical evidence (logs, events, metrics) to determine why the failure happened.

1. **Inspect Kubernetes Pod Events:**
   ```bash
   kubectl describe pod <pod-name> -n <namespace>
   ```
   *Look at the `Events:` section at the bottom for:*
   * `OOMKilled` (Exit Code 137)
   * `Liveness probe failed` (Container unresponsive)
   * `FailedScheduling` (Insufficient CPU/Memory on nodes)

2. **Inspect Current & Previous Container Logs:**
   ```bash
   # Current instance logs
   kubectl logs <pod-name> -n <namespace> --tail=100

   # CRITICAL: Logs from the container instance right BEFORE it crashed
   kubectl logs <pod-name> -n <namespace> --previous --tail=100
   ```

3. **Check Resource Consumption:**
   ```bash
   kubectl top pod <pod-name> -n <namespace>
   kubectl top nodes
   ```

4. **Verify Internal Network & DNS:**
   ```bash
   # Run a temporary debug pod to test service DNS resolution
   kubectl run net-debug --rm -i --tty --image=busybox -- nslookup <service-name>.<namespace>.svc.cluster.local
   ```

---

### Step 4: Permanent Resolution & Recovery
**Goal:** Apply the fix safely using Kubernetes declarative configs.

1. **Apply the updated manifest or patch:**
   ```bash
   kubectl apply -f deployment.yaml
   ```
2. **Monitor the rolling update in real-time:**
   ```bash
   kubectl rollout status deployment/<deployment-name> -n <namespace>
   ```

---

### Step 5: Post-Mortem & Hardening
1. Adjust container `resources.requests` and `resources.limits`.
2. Configure **HorizontalPodAutoscaler (HPA)** for dynamic traffic.
3. Configure **PodDisruptionBudgets (PDB)** to prevent node drains from causing downtime.
4. Add alerts in Prometheus/Grafana for early warning signs (e.g., memory usage > 85%).

---

## 2. Real-World Case Study & Walkthrough

### 🚨 Scenario: E-Commerce Checkout Downtime during Peak Sale
* **Service:** `payment-service` in namespace `prod-checkout`.
* **Symptom:** Customers receive `HTTP 502 Bad Gateway` at checkout. `PagerDuty` triggers an emergency alert.

---

### Step-by-Step Resolution

#### 1. Initial Triage
The SRE engineer checks the status of the `payment-service` pods:
```bash
kubectl get pods -n prod-checkout -l app=payment-service
```
**Output:**
```text
NAME                               READY   STATUS             RESTARTS   AGE
payment-service-6d4b5c7d8-2x9zp    0/1     CrashLoopBackOff   6 (2m ago) 12m
payment-service-6d4b5c7d8-8f1kl    1/1     Running            0          12m
payment-service-6d4b5c7d8-m7n4q    0/1     OOMKilled          0          1m
```

#### 2. Deep-Dive Diagnostics
Run `kubectl describe` on one of the failing pods:
```bash
kubectl describe pod payment-service-6d4b5c7d8-2x9zp -n prod-checkout
```
**Key Findings in Output:**
```text
Last State:     Terminated
  Reason:       OOMKilled
  Exit Code:    137
Limits:
  cpu:     500m
  memory:  256Mi
Events:
  Warning  OOMKilled  2m ago  kubelet  Memory cgroup out of memory: Killed process 18293
  Warning  Unhealthy  1m ago  kubelet  Liveness probe failed: HTTP probe failed with statuscode 500
```

Next, inspect the logs of the container instance that crashed right before restarting:
```bash
kubectl logs payment-service-6d4b5c7d8-2x9zp -n prod-checkout --previous --tail=50
```
**Log Output:**
```text
2026-08-01T17:42:10Z [ERROR] ConnectionPoolExhausted: Timeout waiting for idle database connection (active: 100/100).
2026-08-01T17:42:12Z [FATAL] JavaScript heap out of memory: Allocation failed - process ran out of memory.
```

#### 3. Root Cause Identified
1. **Primary Failure (Resource Constraints):** Memory limit (`256Mi`) was insufficient for high-concurrency traffic during the sale, causing Linux cgroups to terminate the container (`Exit Code 137`).
2. **Secondary Failure (Database Connection Leak):** Unclosed connection handles accumulated during high load, exhausting the connection pool (`100/100`), causing requests to queue up in memory until the heap overflowed.

#### 4. Applying the Fix
Update `deployment.yaml` to increase memory limits and reduce database idle timeouts:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-service
  namespace: prod-checkout
spec:
  replicas: 5
  template:
    spec:
      containers:
      - name: payment-api
        image: registry.company.com/payment-api:v2.4.1
        resources:
          requests:
            cpu: "500m"
            memory: "512Mi"
          limits:
            cpu: "1000m"
            memory: "1Gi"
        env:
        - name: DB_MAX_CONNECTIONS
          value: "50"
        - name: DB_IDLE_TIMEOUT_MS
          value: "3000"
```

Apply the deployment and monitor rollout:
```bash
kubectl apply -f deployment.yaml
kubectl rollout status deployment/payment-service -n prod-checkout
```

#### 5. Verification
Check that all pods are healthy and memory usage is within bounds:
```bash
kubectl get pods -n prod-checkout -l app=payment-service
kubectl top pods -n prod-checkout -l app=payment-service
```
**Output:**
```text
NAME                               READY   STATUS    RESTARTS   AGE   MEMORY
payment-service-7f89d9c4b-a1b2c    1/1     Running   0          3m    320Mi
payment-service-7f89d9c4b-d3e4f    1/1     Running   0          3m    315Mi
payment-service-7f89d9c4b-g5h6i    1/1     Running   0          3m    310Mi
```

Service is fully restored and latency returns to normal.


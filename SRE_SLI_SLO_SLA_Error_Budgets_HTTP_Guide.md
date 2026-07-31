# Master SRE Guide: SLIs, SLOs, SLAs, Observability (ELK Stack) & Incident Management

This master guide covers **SLIs/SLOs/SLAs & Error Budgets**, **Full Observability & ELK Stack Engineering**, and **Incident Response, On-Call, RCA & Blameless Postmortems** for enterprise SRE role interviews and production operations.

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

When calculating Availability SLIs, HTTP status codes must be strictly categorized into **Good Events**, **Bad Events (Error Budget Burners)**, and **Excluded Client Events**.

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

#### Kibana / Elastic (KQL) Filter:
```kql
# Good Events Filter
service.name: "autorabit-release-engine" AND http.response.status_code: [200 TO 399]

# Bad Events Filter (Error Budget Burners)
service.name: "autorabit-release-engine" AND http.response.status_code: [500 TO 599]
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

```
                        THE E-L-K STACK PIPELINE
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│     BEATS       │    │    LOGSTASH     │    │  ELASTICSEARCH  │    │     KIBANA      │
│   (Filebeat)    │ ──►│   (Logstash)    │ ──►│ (Elasticsearch) │ ──►│    (Kibana)     │
├─────────────────┤    ├─────────────────┤    ├─────────────────┤    ├─────────────────┤
│ The Data        │    │ The Data        │    │ The Search &    │    │ The Visual      │
│ Collector       │    │ Transformer     │    │ Storage Engine  │    │ Dashboard       │
└─────────────────┘    └─────────────────┘    └─────────────────┘    └─────────────────┘
```

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

## 6. Index Lifecycle Management (ILM) JSON Policy

```json
{
  "policy": {
    "phases": {
      "hot": {
        "actions": { "rollover": { "max_primary_shard_size": "40gb", "max_age": "7d" } }
      },
      "warm": {
        "min_age": "7d",
        "actions": { "shrink": { "number_of_shards": 1 }, "forcemerge": { "max_num_segments": 1 } }
      },
      "cold": {
        "min_age": "30d",
        "actions": { "searchable_snapshot": { "snapshot_repository": "s3_backup" } }
      },
      "delete": { "min_age": "90d", "actions": { "delete": {} } }
    }
  }
}
```

---

## 7. Cluster Troubleshooting (`Red` vs `Yellow` vs `Green`)

- **`Green`**: All primary and replica shards assigned.
- **`Yellow`**: Primary shards healthy, replica shards unassigned (No redundancy).
- **`Red`**: Primary shard missing/corrupted! (Data loss / query failure occurring).

#### Remediation for `Red` Cluster:
```bash
GET /_cluster/health
GET /_cluster/allocation/explain
```
If disk full (`disk.watermark.flood_stage` hit), clear disk space and unlock index:
```json
PUT /<index_name>/_settings
{ "index.blocks.read_only_allow_delete": null }
```

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

## 3. Incident Severity Matrix & SLAs

| Severity Level | Definition / Impact | Target MTTA (Acknowledge) | Target MTTR (Repair) | Communication Frequency |
| :--- | :--- | :--- | :--- | :--- |
| **P1 - Critical** | Total SaaS outage or core data pipeline failure affecting multiple enterprise tenants. | **< 5 minutes** | **< 30 minutes** | Every 15 minutes |
| **P2 - High** | Core feature degraded with no workaround (e.g., Salesforce sync delay > 1 hour). | **< 15 minutes** | **< 2 hours** | Every 30 minutes |
| **P3 - Medium** | Non-critical component degraded, workaround available (e.g., UI log rendering slow). | **< 1 hour** | **< 24 hours** | Every 4 hours |
| **P4 - Low** | Minor bug, cosmetic issue, or low-priority internal alert. | **< 4 hours** | Next Sprint | Business Days |

---

## 4. Root Cause Analysis (RCA): The 5-Whys Framework

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

## 5. Blameless Postmortem Template

```markdown
# Incident Postmortem: [P1] Release Engine Database Exhaustion

**Date**: 2026-08-01 | **Incident Commander**: [SRE Name] | **Severity**: P1  
**Impact**: 1,200 Enterprise Customers experienced HTTP 503 errors for 25 minutes.  
**Error Budget Consumed**: 38% of monthly budget.  

---

## 1. Executive Summary
On August 1, 2026 at 14:10 UTC, the AutoRABIT Release Engine experienced a P1 outage lasting 25 minutes due to database connection pool saturation following a surge in metadata sync requests. Service was restored by executing an automated pod restart and deploying PgBouncer connection pooling configurations.

---

## 2. Preventative Action Items (Jira Tracking)

| Action Item | Type | Owner | Priority | Target Date | Jira Ticket |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Deploy PgBouncer in front of RDS via Terraform | Prevention | SRE Team | P0 (Blocker) | 3 Days | `ENG-1042` |
| Add `statement_timeout = 5s` to Postgres config | Mitigation | DBA Team | P1 | 5 Days | `ENG-1043` |
| Update PagerDuty runbook with RDS Grafana links | Process | SRE Team | P2 | 7 Days | `ENG-1044` |
```

---

# SECTION 4: STAR Interview Model Answers

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

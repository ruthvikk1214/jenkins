# Mock Interview – Senior SRE (AutoRabit) with Suggestions

## 1. Introduction & Warm‑up
**Question:** "Tell me about yourself, your current role, and why you're interested in a Senior SRE position at AutoRabit."
**User Answer:**
```
Hello , I'm Ruthvekh a Devops Engineer at Kyndryl . I have 3 years of experience as a devops engineer and i have been working on k8s, docker, terraform, cicd. I also have experience in SRE practices like ELK stack, troubleshooting, defining SLI/SLO/SLA , error budgeting, devsecops practices, linux os hardening practices, and implementing CIS benchmarks
```
**Suggestions:**
- Add a brief impact statement (e.g., *"Managed 800+ Linux servers with 99.9% uptime"*).
- Mention why AutoRabit specifically (its Salesforce DevOps platform, scaling challenges, etc.).
- Structure answer: *who you are → what you’ve achieved (metrics) → why this role*.

---
## 2. SLI / SLO / SLA / Error Budget
**Question:** "Explain the relationship between SLIs, SLOs, SLAs, and Error Budgets. Can you give me a real example from your experience where you defined these, and what happened when the error budget was being consumed too fast?"
**User Answer:**
```
sli-> the service level indicaiton. this measures the performance of the services. 
ex: 99.9% of the api requests are success, only 0.1% are failed
slo->service level objective. this is the target the teams set internally. generally this would be more than the sla just for our safety
ex: if sla is agreed on 99% over the period of a month , then the slo would be 99.5%
sla-> the agreement b/w service provider and client. this describes the financial/ legal penalties if slo is missed
ex: if sla is missed by 0.5% , lets say the client gets 15% of the contract credit back

relation b/w sli,sla,slo is that each is dependent on other. each parameter is built on another.
sli-> would be defined in prometheus, grafana in the monitoring rules
slo-> datadog, aws clouds
sla-> agreement made b/w client and service provider
```
**Suggestions:**
- Define **SLI** as the **raw metric** (e.g., *"API success rate"*), not the target.
- **SLO** is the target on that metric (e.g., *"99.9% success over 30‑day window"*).
- Include a concrete error‑budget example from your work (e.g., *"Error budget = 0.1% → 43 min/month; we burned 70 % in 2 days, triggered a fast‑burn alert, rolled back, and restored the budget"*).
- Mention fast‑burn vs slow‑burn monitoring.

---
## 3. ECS vs EKS & OOMKilled
**Question:** "You need to deploy a microservices application on AWS. Walk me through how you'd decide between ECS (Fargate) vs EKS. What factors would influence your choice? And once deployed on EKS, how would you handle a scenario where pods are getting OOMKilled intermittently in production?"
**User Answer:**
```
firstly , for any microservice based application, EKS is the best choice. because of its portability, it can be run on any cloud provider or even on‑prem. eks is best for long‑time running apps as it is reliable, and we can multiple 3rd party monitoring tools. but , fargate is recommended only when the traffic is variable, and burst workloads are expected and we can only use aws cloudwatch as monitoring because of being native aws service. however it gives the serverless feel and we pay for container per second. so it is costly than ec2.
```
**User Answer (OOMKilled):**
```
... pods are getting OOMKILLED, firstly i will check  if there is traffic spike, monitoring dashboards , cluster health, node health. Then i will be checking the pods which are getting OOMKILLED and inspect the logs. I will try to understand which event has triggered OOMKILLED. Mostly we see OOMKILLED because of VPC CNI IP exhaustion, or free cpu / memory not left on nodes, thus resulting in failure of creating new pods by HPA. if the cpu/memory is not sufficient for pods to handle the traffic, i wiill increase the resource limits
```
**Suggestions:**
- Clarify decision matrix (team expertise, workload complexity, operational overhead, cost). Mention that **Fargate works with both ECS and EKS**.
- For **OOMKilled**, stress that it is a **memory limit** issue, not IP exhaustion or CPU. Suggest:
  1. `kubectl describe pod <pod>` → confirm exit code 137.
  2. Check `kubectl top pod` vs `resources.limits.memory`.
  3. Increase memory limit or enable **VPA**.
  4. Investigate memory leaks with profiling if limit increase isn’t enough.
- Mention using **PodDisruptionBudget** and **Cluster Autoscaler** to keep capacity.

---
## 4. Kubernetes Networking
**Question:** "Explain how Kubernetes networking works inside an EKS cluster. How does a request from the internet reach a specific pod? Walk me through the full traffic flow — from Route53 all the way to the container process."
**User Answer:**
```
in kubernetes, networking works like below
every pod gets a unique ip address from the VPC CIDR
pods on same node communicate via bridge
pods on different nodes communicate via CNI plugin
clusterIP exposes the application internally
nodeport exposes the application based on a port on every node(30000-32600)
loadbalancer creates a load balacer(app/network) in the cloud provider
when a request is made to a a DNS the request goes like nelow

DNS->app loadbalancer->ingress->k8s service->pod->application
```
**Suggestions:**
- Mention **NodePort range is 30000‑32767**.
- Detail each hop:
  1. **Route 53** → resolves to ALB DNS (or NLB) via an Alias record.
  2. **ALB (AWS Load Balancer Controller)** terminates TLS, applies listener rules, forwards to **Target Group** (IP‑mode → pod ENI, or instance‑mode → node IP).
  3. **kube‑proxy** (iptables/IPVS) performs DNAT from Service IP to pod IP.
  4. **CNI (aws‑vpc‑cni)** provides each pod a VPC IP, enabling direct pod‑to‑pod traffic.
- Mention **Ingress** is optional when using Service of type LoadBalancer directly.

---
## 5. ELK Logging Pipeline Design
**Question:** "At AutoRabit, we use the ELK stack for all our observability. Tell me — how would you design a centralized logging pipeline for 50+ microservices running on EKS? Walk me through the architecture, and explain how you'd handle log parsing, high‑volume ingestion, and setting up meaningful alerts in Kibana."
**Model Answer (your):**
```
I will be creating a pipleline , with filebeat/fluentbit as log collectors from /var/log/containers from every node. we deploy fluentbit as daemonset with minimal memory footprint.then in next stage we use logstash to convert the raw log data into organized indexed data. in the next stage we store the data in elasticsearch and use ILM policies to move the data across tiers based on duration. we use the hot tier for log writing and storing logs of 30 days or 50GB whichever comes first.warm for 30-90 days, cold for 90-360. this is very cost effective. next we use kibana to use the logs from elastic search to create visual dashboards out of the log data
```
**Suggestions:**
- Use **Fluent Bit** (DaemonSet) → **Kafka** (buffer) → **Logstash** (parsing) → **Elasticsearch** (ILM). Kafka decouples producers from consumers for high‑volume spikes.
- **Parsing:** JSON logs → `json` filter; unstructured logs → `grok` patterns; add **Kubernetes metadata** (`kubernetes` filter) to enrich logs with pod, namespace, labels.
- **Kibana alerts:** Use **Watcher** or **Alerting** to trigger on error‑rate spikes, missing logs, or latency thresholds.
- Include **index templates** for consistent mappings and **ingest pipelines** for field extraction.

---
## 6. Terraform Structure & State Management
**Question:** "You're writing Terraform to provision a production VPC, EKS cluster, and RDS PostgreSQL — all in a single codebase. How do you structure your Terraform code for reusability across dev, staging, and prod? Also, explain what a Terraform state file is, why it's critical, and what happens if two engineers run `terraform apply` at the same time."
**User Answer:**
```
If i am developing the code for a production vpc, eks, rds in a single code base, then i will create each component in separate folders. I will write modules for each component, and parameterize the important variables like environment, sg names, resource limits in eks, so that the module can be reused across all environments. A terraform state file is a json file which is used to track the infra created/managed by terraform. It is critical because it holds the current configuration created using terraform, so whenever a new resource is being created , the first thing terraform does it to check the current configuration using state file. That is the reason no one should modify the state file manually , and also it is preffered to store in cloud securely. if 2 engineers run terraform apply at the same time, the 2nd engineer code execution might overrite the 1st engineers code. i mean whatever the resources created by the 1st engineer will be created and will be updated in the state file. but when 2nd engineer runs apply , this will overrite the configuration made by 1st engineer. thus the state fill will get corrupted. this may lead to a production incident. to prevent this, we need to use dynamodb state locking so that when a person is performing something, the state file will be locked and any other person tries to run anything, it will be hold, and they will get a notification that the state is locked by 1 engineer
```
**Suggestions:**
- Show a typical folder layout (`modules/`, `environments/dev/`, `environments/prod/`).
- Emphasize **backend "s3"** with **DynamoDB lock table**.
- Clarify that without locking the two runs would **read the same state**, then both would try to apply → duplicate resources or drift, not simply overwrite.
- Mention **remote CI execution** to enforce single‑owner apply.

---
## 7. AWS‑Native CI/CD Pipeline Design
**Question:** "At AutoRabit, we use AWS‑native CI/CD — CodePipeline, CodeBuild, and CodeDeploy. You've mostly used Jenkins and GitHub Actions. Tell me — how would you design a CI/CD pipeline using AWS‑native tools to build, test, scan, and deploy a containerized application to EKS? And how does CodeDeploy handle rolling vs blue/green deployments on EKS?"
**User Answer:**
```
... (see previous answer) ...
```
**Suggestions:**
- **CodeBuild** runs in a **managed Docker container**, not a fresh EC2 instance. Reference a `buildspec.yml`.
- For **EKS deployments**, CodeDeploy is not used; instead a **CodeBuild step runs `kubectl`/`helm`** or uses **ArgoCD**.
- Clarify that **CodeDeploy** natively supports **ECS** (blue/green) and **EC2**, not EKS.
- Explain **rolling update** in Kubernetes (maxSurge/maxUnavailable) vs **blue/green** via separate Deployments or **Argo Rollouts**.

---
## 8. CIS Benchmarks & Hardening
**Question:** "What are CIS benchmarks, and how would you implement CIS Level 2 hardening on Ubuntu EC2 instances at scale? Give me specific examples of controls you'd enforce. Also, how would you ensure ongoing compliance — not just a one‑time hardening?"
**User Answer:**
```
... (see previous answer) ...
```
**Suggestions:**
- Add **ongoing compliance** tools: **AWS Inspector**, **OpenSCAP**, **SSM State Manager**, **AWS Config Rules**.
- Mention **Automated remediation** via SSM associations.
- Provide more Level‑2 controls (e.g., disable USB, enforce MFA for IAM, enable auditd with specific rules, enforce password complexity, configure `fs.protected_hardlinks` and `fs.protected_symlinks`).

---
## 9. Incident Response & Postmortem
**Question:** "It's 2 AM. You get paged — the checkout API is returning 500 errors for 30% of requests. Walk me through your exact incident response process from the moment you get the alert to the postmortem you write 3 days later. Be specific about what tools you use, what you check first, how you communicate, and what a good postmortem looks like."
**User Answer:**
```
... (see previous answer) ...
```
**Suggestions:**
- Replace **"silence"** with **"acknowledge"** the alert.
- Open a **Slack war‑room**, post an initial incident message with severity (SEV 1).
- Update a **status page** for customers.
- Use **error‑budget burn‑rate alerts** to trigger the incident.
- Postmortem template should include:
  - Timeline with timestamps.
  - Root‑cause analysis.
  - Action items with owners and due dates.
  - Metrics before/after.
- Emphasize “blameless” but also **preventive actions**.

---
## 10. Cost‑Optimized Multi‑Region EKS Design
**Question:** "Design a cost‑optimized, multi‑region deployment for an EKS‑based service while meeting latency and resilience SLAs. Include AWS services, autoscaling, and monitoring."
**Model Answer:** (see earlier detailed design).
**Suggestions:**
- Highlight **Route 53 latency‑based routing** with health checks.
- Use **AWS Global Accelerator** for ultra‑low latency (optional).
- Add **cross‑region CloudWatch dashboards**.
- Mention **cost allocation tags** for per‑region spend tracking.

---
## 11. Auto‑Remediation for CrashLoopBackOff (Pending)
**Question:** "Imagine a pod repeatedly crashes with CrashLoopBackOff because a downstream API is unavailable. How would you design an auto‑remediation loop that detects the issue, isolates the pod, and attempts recovery without manual intervention? Mention AWS services, Kubernetes objects, and any alerting or IAM considerations."
**Answer:** *Pending – we can discuss this next.*

---
*End of compiled mock interview with suggestions.*

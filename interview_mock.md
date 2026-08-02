# Mock Interview – Senior SRE (AutoRabit)

## 1. Introduction & Warm‑up
**Question:** "Tell me about yourself, your current role, and why you're interested in a Senior SRE position at AutoRabit."
**User Answer:**
```
Hello , I'm Ruthvekh a Devops Engineer at Kyndryl . I have 3 years of experience as a devops engineer and i have been working on k8s, docker, terraform, cicd. I also have experience in SRE practices like ELK stack, troubleshooting, defining SLI/SLO/SLA , error budgeting, devsecops practices, linux os hardening practices, and implementing CIS benchmarks
```

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

## 3. ECS vs EKS & OOMKilled
**Question:** "You need to deploy a microservices application on AWS. Walk me through how you'd decide between ECS (Fargate) vs EKS. What factors would influence your choice? And once deployed on EKS, how would you handle a scenario where pods are getting OOMKilled intermittently in production?"
**User Answer:**
```
firstly , for any microservice based application, EKS is the best choice. because of its portability, it can be run on any cloud provider or even on-prem. eks is best for long-time running apps as it is reliable, and we can multiple 3rd party monitoring tools. but , fargate is recommended only when the traffic is variable, and burst workloads are expected and we can only use aws cloudwatch as monitoring because of being native aws service. however it gives the serverless feel and we pay for container per second. so it is costly than ec2.
```
**User Answer (continued – OOMKilled):**
```
firstly , for any microservice based application, EKS is the best choice. because of its portability, it can be run on any cloud provider or even on-prem. eks is best for long-time running apps as it is reliable, and we can multiple 3rd party monitoring tools. but , fargate is recommended only when the traffic is variable, and burst workloads are expected and we can only use aws cloudwatch as monitoring because of being native aws service. however it gives the serverless feel and we pay for container per second. so it is costly than ec2. and in the scenario where the pods are getting OOMKILLED, firstly i will check  if there is traffic spike, monitoring dashboards , cluster health, node health. Then i will be checking the pods which are getting OOMKILLED and inspect the logs. I will try to understand which event has triggered OOMKILLED. Mostly we see OOMKILLED because of VPC CNI IP exhaustion, or free cpu / memory not left on nodes, thus resulting in failure of creating new pods by HPA. if the cpu/memory is not sufficient for pods to handle the traffic, i wiill increase the resource limits
```

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

## 5. ELK Logging Pipeline Design
**Question:** "At AutoRabit, we use the ELK stack for all our observability. Tell me — how would you design a centralized logging pipeline for 50+ microservices running on EKS? Walk me through the architecture, and explain how you'd handle log parsing, high-volume ingestion, and setting up meaningful alerts in Kibana."
**Model Answer:**
```
I will be creating a pipleline , with filebeat/fluentbit as log collectors from /var/log/containers from every node. we deploy fluentbit as daemonset with minimal memory footprint.then in next stage we use logstash to convert the raw log data into organized indexed data. in the next stage we store the data in elasticsearch and use ILM policies to move the data across tiers based on duration. we use the hot tier for log writing and storing logs of 30 days or 50GB whichever comes first.warm for 30-90 days, cold for 90-360. this is very cost effective. next we use kibana to use the logs from elastic search to create visual dashboards out of the log data
```

## 6. Terraform Structure & State Management
**Question:** "You're writing Terraform to provision a production VPC, EKS cluster, and RDS PostgreSQL — all in a single codebase. How do you structure your Terraform code for reusability across dev, staging, and prod? Also, explain what a Terraform state file is, why it's critical, and what happens if two engineers run `terraform apply` at the same time."
**User Answer:**
```
If i am developing the code for a production vpc, eks, rds in a single code base, then i will create each component in separate folders. I will write modules for each component, and parameterize the important variables like environment, sg names, resource limits in eks, so that the module can be reused across all environments. A terraform state file is a json file which is used to track the infra created/managed by terraform. It is critical because it holds the current configuration created using terraform, so whenever a new resource is being created , the first thing terraform does it to check the current configuration using state file. That is the reason no one should modify the state file manually , and also it is preffered to store in cloud securely. if 2 engineers run terraform apply at the same time, the 2nd engineer code execution might overrite the 1st engineers code. i mean whatever the resources created by the 1st engineer will be created and will be updated in the state file. but when 2nd engineer runs apply , this will overrite the configuration made by 1st engineer. thus the state fill will get corrupted. this may lead to a production incident. to prevent this, we need to use dynamodb state locking so that when a person is performing something, the state file will be locked and any other person tries to run anything, it will be hold, and they will get a notification that the state is locked by 1 engineer
```

## 7. AWS‑Native CI/CD Pipeline Design
**Question:** "At AutoRabit, we use AWS‑native CI/CD — CodePipeline, CodeBuild, and CodeDeploy. You've mostly used Jenkins and GitHub Actions. Tell me — how would you design a CI/CD pipeline using AWS‑native tools to build, test, scan, and deploy a containerized application to EKS? And how does CodeDeploy handle rolling vs blue/green deployments on EKS?"
**User Answer:**
```
Okay, in the AWS native ci/cd tools, this is how i would design a CI/CD pipeline
whenever a new push is done to github, the codepipeline use webhook to get triggered whenever a push is done. the next thing it does it to initiate the CI using codebuild. in this CI process the codebuild takes a fresh ec2 instance and does all the CI part -- code checkout done by codepipeline into a zip file stored in S3 bucker, the codebuild will fetch it, and start installing dependencies and build the app , and perform all sorts of testing like unit , and addin sonarqube and trivy to scan the code for quality and trivy to scan images for vulnerabilities. after CI process is done, codebuild creates a zip file into the same S3 bucket and alerts codepipeline that build(CI ) is done. next the codepipeline triggers codedeploy to perform the deployment/delivery based on our preference or requirement. Continuous deployment does not need any manual interntion to create deployment, where as continuous delivery needs it. now coming to deployment strategies, the blue-green is a strategy where we create a deployment similar to the one in use, only difference is it is running the newer version of the app. the blue indicates the current stable deployment, green indicates newer version of app containing deployment. the traffic is completely routed to the green deployment , and monitoring is done. whethere the deployment fails or end users are facing any issues/bugs/lag. if green deployment is not success then the entire traffic is routed back to blue deployment(stable). generally blue-green deployment is very costly, we need to create another deployment of same configuration . coming to canary deployment, we route a less number of traffic to the new deployment and monitor continuously . we gradually increase the traffic percentage based on the success rate. this is mostly preffered. because low infra creation, cost effective
```

## 8. CIS Benchmarks & Hardening
**Question:** "What are CIS benchmarks, and how would you implement CIS Level 2 hardening on Ubuntu EC2 instances at scale? Give me specific examples of controls you'd enforce. Also, how would you ensure ongoing compliance — not just a one‑time hardening?"
**User Answer:**
```
Center for internet security benchmarks are used to harden os like ubuntu, rhel, amazon linux , to improve security and prevent attacks. these may affect performance of the infrastructure. in order to implement cis level2 at scale, we have few options. either use hardened images(available over cloud or os providers) , or else we can configure the instances using ansible jobs, we can write ansible scripts to implement the settings across environments at a time. some of the cis settings are
1. set permitrootlogin to no
2.lockout user after 5 login failure.
3.enable auditd on all servers so that it can track all activities on the server, and at secure files too
4.modify kernel parameters like disable ipv4 forwarding and syncflood protection enabled
5.for filesystems and files, setting umask 027. and also mounting /tmp , /var filesystems with nosuid noexec etc
```

## 9. Incident Response & Postmortem
**Question:** "It's 2 AM. You get paged — the checkout API is returning 500 errors for 30% of requests. Walk me through your exact incident response process from the moment you get the alert to the postmortem you write 3 days later. Be specific about what tools you use, what you check first, how you communicate, and what a good postmortem looks like."
**User Answer:**
```
this is the approach i follow
I silence down the alert first and let the system know that a responder is addressing the issue
Check the grafana, prometheus, kibana, dashboards to get a better understanding of the issue, what number of outage is happening, percentage of requests getting failed, what is the response http code
Then i will open up an incident and takeup as the Incident Commander
I will try to understand the blast radius of the issue , what are getting affectted
Next i will check what has changed in the last deployment, and what caused the issue:-- 1)if this is related to the application code or a new feature, then roll back to previous stable version, 2)if this is related to traffic spike like  memory / cpu usage issue( based on the logs of pods either OOMKILLED), i will modify the resource limits of pods , 3) or else if ths is related to db worker nodes getting stuck then i will try to restart the deployment to make the cluster healthy again
based on the fix i perform i will continue to monitor the deploment for 1-2 hours and make sure the 5xx error are within the slo budget or 0 , if eveything is fine i will update the Incident and close it
then , the next day we will draft blameless postmortem and document everything that caused the outage, the impact occured, the fix we did including timestamps
after that we will conduct a meeting on this issue, and discuss about countermeasure to prevent this from happening again
```

## 10. Cost‑Optimized Multi‑Region EKS Design
**Question:** "Design a cost‑optimized, multi‑region deployment for an EKS‑based service while meeting latency and resilience SLAs. Include AWS services, autoscaling, and monitoring."
**Model Answer:** (see above in the interview)
```
[Full design described in previous answer – includes architecture overview, autoscaling, cost‑reduction techniques, monitoring, resilience, Terraform/ArgoCD sketch, flow diagram, TL;DR]
```

## 11. Auto‑Remediation for CrashLoopBackOff (Pending)
**Question:** "Imagine a pod repeatedly crashes with CrashLoopBackOff because a downstream API is unavailable. How would you design an auto‑remediation loop that detects the issue, isolates the pod, and attempts recovery without manual intervention? Mention AWS services, Kubernetes objects, and any alerting or IAM considerations."
**Answer:** *Pending – to be discussed in the next session.*

---
*End of compiled mock interview.*

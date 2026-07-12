# Jenkins, SonarQube, and EKS Integration Runbook

This runbook summarizes all the troubleshooting, configuration fixes, and setup steps performed for the Jenkins Controller, Jenkins Agent, SonarQube, and EKS deployment pipeline.

---

## 1. Jenkins Master Bootstrap Fixes

### LVM Disk Resizing Fix
* **Issue:** The bootstrap script (`user_data.sh`) was running `pvcreate /dev/xvda` and `vgextend RootVG /dev/xvda` on the active root disk, causing filesystem metadata conflicts that crashed the EC2 instances.
* **Fix:** Updated the LVM resize section to safely grow the existing partition and physical volume dynamically, supporting both Xen (`/dev/xvda`) and Nitro (`/dev/nvme0n1`) layouts:
  ```bash
  if [ -b /dev/nvme0n1 ]; then
      growpart /dev/nvme0n1 4 || true
      pvresize /dev/nvme0n1p4 || true
  elif [ -b /dev/xvda ]; then
      growpart /dev/xvda 4 || true
      pvresize /dev/xvda4 || true
  fi
  lvextend -r -l +100%FREE /dev/RootVG/varVol || true
  ```

### Package Conflict & /boot Space Failures
* **Issues:**
  1. `dnf install -y docker` was causing package conflicts with the official Docker CE repository installed later.
  2. `dnf update -y` attempted a full kernel upgrade, which ran out of space in the small `/boot` partition of the AMI, halting execution.
* **Fixes:**
  * Removed `docker` and `awscli` packages from the initial `dnf install` block.
  * Removed the `dnf update -y` command to prevent kernel installation failures.

---

## 2. Terraform Infrastructure Configuration

### User Data Replacement Policy
* **Update:** Added `user_data_replace_on_change = true` to the Jenkins Master instance in main.tf so that modifying the bootstrap script automatically triggers instance recreation.

### SonarQube Workspace Setup (Marketplace Workaround)
* **Issue:** The AWS Marketplace AMI required a terms subscription agreement which failed due to a billing/payment method issue on the AWS account.
* **Fix:** 
  1. Switched the AMI data source in `data.tf` to Canonical's official free base Ubuntu 24.04.
  2. Created `sonar.sh` to automatically install Java 17 and SonarQube CE with an embedded database on first boot.
  3. Created the `sonarqube` security group in `10-sg`, mapped it in `20-sg-rules`, and opened ports `9000` (Web UI) and `22` (SSH).

### Route53 Reference and Count Fixes
* **Issue:** Route53 records for `jenkins_agent` and `sonarqube` failed to compile because they referenced list resources (`count` attributes) without list indexes.
* **Fix:** Corrected `r53.tf` to use list index `[0]` and matching `count` conditionals:
  ```hcl
  records = [aws_instance.jenkins_agent[0].public_ip]
  ```

---

## 3. Jenkins & EKS Integration Setup

### Jenkins Agent IAM Authentication
* **Update:** Created `roboshop-dev-jenkins-role` with `AdministratorAccess` and attached it via an IAM Instance Profile to both the Jenkins Master and Jenkins Agent EC2 instances. This grants passwordless AWS API access.

### Jenkins Pipeline (Jenkinsfile) Updates
1. **Dynamic Sonar-Scanner Installation:** Added a fallback shell installer to automatically download and run the `sonar-scanner` tool on-the-fly if it is not pre-installed on the agent:
   ```groovy
   if ! command -v sonar-scanner &> /dev/null; then
       curl -sSLo /tmp/sonar-scanner.zip https://binaries.sonarsource.com/Distribution/sonar-scanner-cli/sonar-scanner-cli-5.0.1.3006-linux.zip
       unzip -q -o /tmp/sonar-scanner.zip -d /tmp/
       export PATH="/tmp/sonar-scanner-5.0.1.3006-linux/bin:$PATH"
   fi
   ```
2. **Kubernetes Configuration (`kubeconfig`):** Added a command to authenticate `kubectl` to the EKS cluster inside the pipeline before deploying:
   ```groovy
   aws eks update-kubeconfig --name roboshop --region us-east-1
   ```

---

## 4. Post-Deployment Cluster Authorization (Manual Step)

Because the EKS cluster was created manually, only the creator credentials had access to the Kubernetes API. The Jenkins Agent role was rejected with a credentials required error.

### Access Authorization Steps:
1. **Log in to the EKS Creator AWS session on the Server:**
   ```bash
   aws configure # Enter EKS creator keys
   aws eks update-kubeconfig --name roboshop --region us-east-1
   ```
2. **Map the Jenkins IAM Role in Kubernetes:**
   * Export the ConfigMap:
     ```bash
     kubectl get configmap aws-auth -n kube-system -o yaml > aws-auth.yaml
     ```
   * Open `aws-auth.yaml` and add the Jenkins Agent Role under `mapRoles: |`:
     ```yaml
     - rolearn: arn:aws:iam::628087992516:role/roboshop-dev-jenkins-role
       username: jenkins
       groups:
         - system:masters
     ```
   * Apply the ConfigMap:
     ```bash
     kubectl apply -f aws-auth.yaml -n kube-system
     ```
3. **Create the Namespace:**
   ```bash
   kubectl create namespace roboshop
   ```

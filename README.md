WordPress on Amazon EKS: Helm, Autoscaling & Load Testing

Deployed a containerized WordPress + MySQL application to Amazon EKS with Helm, configured Horizontal Pod Autoscaling, and load-tested it until it broke, then traced the failure to its root cause.

<img width="3510" height="2218" alt="WordPress on Amazon EKS" src="https://github.com/user-attachments/assets/fa7169f1-3b94-4bec-8949-2b061758cb53" />


Overview

This hands-on project takes WordPress from a single server to a Kubernetes cluster on AWS. The cluster runs on three worker nodes in private subnets, the app is packaged as a Helm chart, and an autoscaler adds pods when CPU climbs. A 100-user load test then pushed it past its limits.

The original lab guide had several gaps that would have stopped the deployment from working. Fixing them, and diagnosing what happened under load, became the most valuable part of the project.

Highlights

Provisioned an EKS cluster (Kubernetes 1.34) with worker nodes in private subnets, built through CloudFormation by eksctl
Packaged WordPress as a Helm chart and added the MySQL backend the guide was missing, with passwords stored in a Kubernetes Secret
Restricted the public load balancer to a single allow-listed IP
Configured a Horizontal Pod Autoscaler that scaled WordPress 1 → 3 pods under load
Diagnosed failed requests during the load test down to pods being OOMKilled (exit code 137)
Tech stack
Area	Tools
Cloud	Amazon EKS, EC2 (t3.small managed node group), VPC, NAT gateway, Classic Load Balancer, CloudFormation, IAM
Kubernetes	Deployments, Services, Secrets, Horizontal Pod Autoscaler, metrics-server
Packaging	Helm 4
Application	WordPress, MySQL 8.4
Testing	Siege (HTTP load testing)
CLI tooling	AWS CLI, eksctl, kubectl, PowerShell on Windows
Architecture
Layer	Component	Purpose
Provisioning	eksctl → CloudFormation	Two stacks: the cluster (VPC, control plane, IAM roles) and the node group
Control plane	Amazon EKS (AWS-managed)	Kubernetes API and scheduling
Public subnets	Classic Load Balancer	Public entry point, restricted to one IP (/32)
Public subnets	NAT gateway	Outbound-only internet access so private nodes can pull images
Private subnets	3 × t3.small nodes	Run all workloads; no public IP addresses
Workloads	WordPress pods (Helm release)	The web application, scaled 1–5 replicas by the HPA
Workloads	MySQL pod + ClusterIP Service	Database, reachable only inside the cluster
Scaling	metrics-server → HPA	Reads pod CPU every ~15 s and adjusts replica count
How I built it
1. Tooling and access

Installed eksctl and Helm with winget (which verifies installer hashes), and configured the AWS CLI with a dedicated IAM user, not the root account.

2. Created the cluster
powershell
eksctl create cluster --name enterprise-cluster --region us-east-1 `
  --nodegroup-name standard-workers --node-type t3.small `
  --nodes 3 --nodes-min 3 --nodes-max 6 --managed --node-private-networking

--node-private-networking keeps the worker nodes off the public internet.

3. Built the Helm chart

Generated a chart with helm create my-microservice and edited values.yaml:

yaml
image:
  repository: wordpress
  tag: "latest"

service:
  type: LoadBalancer
  port: 80
  loadBalancerSourceRanges:
    - "YOUR-IP/32"          # only this IP can reach the site

resources:
  requests: { cpu: 300m, memory: 256Mi }
  limits:   { cpu: 500m, memory: 512Mi }

livenessProbe:
  httpGet: { path: /, port: http }
  initialDelaySeconds: 30
  timeoutSeconds: 5          # default of 1 s killed healthy-but-busy pods

env:
  - name: WORDPRESS_DB_HOST
    value: mysql             # the MySQL Service name
  - name: WORDPRESS_DB_PASSWORD
    valueFrom:
      secretKeyRef: { name: mysql-secret, key: user-password }

I also edited the chart's deployment.yaml and service.yaml templates so these new env and loadBalancerSourceRanges values are actually rendered.

4. Added the database

Generated random passwords in memory and stored them only in a Kubernetes Secret, so no password was ever written to a file:

powershell
$rootPw = -join ((48..57)+(65..90)+(97..122) | Get-Random -Count 24 | % {[char]$_})
$userPw = -join ((48..57)+(65..90)+(97..122) | Get-Random -Count 24 | % {[char]$_})
kubectl create secret generic mysql-secret `
  --from-literal=root-password=$rootPw --from-literal=user-password=$userPw

MySQL runs as its own Deployment with an internal-only Service, and reads its passwords from that Secret:

yaml
containers:
  - name: mysql
    image: mysql:8.4                       # pinned version
    env:
      - name: MYSQL_DATABASE
        value: wordpress_db
      - name: MYSQL_USER
        value: wordpress_user
      - name: MYSQL_PASSWORD
        valueFrom:
          secretKeyRef: { name: mysql-secret, key: user-password }
5. Deployed and verified
powershell
helm install my-microservice ./my-microservice
kubectl get pods
kubectl get svc my-microservice
6. Configured autoscaling
yaml
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: wordpress-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-microservice      # the guide said "wordpress", which doesn't exist
  minReplicas: 1
  maxReplicas: 5
  targetCPUUtilizationPercentage: 50
7. Load tested

Ran Siege inside the cluster against the internal Service name, since the public load balancer only accepts one IP:

powershell
kubectl run siege --image=jstarcher/siege --restart=Never --command -- `
  /bin/sh -c "siege -c 100 -t 5m http://my-microservice/"
Gaps I fixed in the original guide
Problem in the guide	What I did
WordPress deployed with no database	Added a MySQL Deployment, an internal Service, and a Secret for credentials
Load balancer open to the whole internet	Added loadBalancerSourceRanges to allow one IP only
Worker nodes given public IPs	Used --node-private-networking
HPA targeted a Deployment named wordpress, which doesn't exist	Pointed it at my-microservice
Siege targeted the external address, which the IP allow-list blocks	Targeted the in-cluster Service name instead
Health checks timed out after 1 second	Raised to 5 seconds so busy pods aren't restarted
t3.medium nodes, not allowed on the AWS Free plan	Switched to t3.small
Commands written for Git Bash, and kubectl never installed	Adapted everything to PowerShell
Troubleshooting
1. Cluster created, but no worker nodes
Symptom: eksctl timed out after ~25 minutes; the console showed the cluster Active with 0 nodes.
Diagnosis: The node group was stuck in CREATING with no health issues, then rolled back. Its CloudFormation events held the reason.
Root cause: InvalidParameterCombination: The specified instance type is not eligible for Free Tier.
Fix: Deleted the partial cluster with eksctl (so the VPC and NAT gateway were removed too) and recreated it with t3.small.
2. Autoscaling worked, but most requests failed

HPA behaviour during the test

Time	CPU (vs. 50% target)	Replicas	What happened
Before	5–6%	1	Idle
+15 s	55%	1	Within the HPA's 10% tolerance, so no action
+60 s	111%	1	Overloaded
+75 s	14%	3	Scaled out: ⌈1 × 111 / 50⌉ = 3

Siege results (5 minutes, 100 concurrent users)

Metric	Result
Successful transactions	242
Failed transactions	466
Availability	34.18%
Average response time	7.39 s
Longest transaction	30.90 s

Diagnosis

CPU stayed low after scaling, yet requests were timing out, so CPU wasn't the bottleneck. That's also why the HPA stopped at 3 pods.
kubectl describe pod on the original pod showed:
  Last State:  Terminated
    Reason:    OOMKilled
    Exit Code: 137
Its events showed liveness and readiness probes timing out before the kill, then connection refused while it restarted.

Root cause: WordPress's Apache server starts a separate worker process per concurrent visitor. With 100 visitors arriving at once, memory hit the pod's 512 Mi limit and the container was killed.

Lessons

Autoscaling only reacts to the metric it watches. A memory-bound app needs memory limits sized from load tests, or bounded worker counts.
Liveness probes should catch frozen apps, not busy ones. Strict liveness settings restart pods exactly when they're needed most.
Next fix: cap Apache's workers per pod (e.g., MaxRequestWorkers) so extra requests queue instead of crashing the pod. That makes CPU the bottleneck, which the HPA can act on.
Screenshots

Cluster overview: Active, Kubernetes 1.34, 3 nodes, 0 health issues 

<img width="1602" height="697" alt="01-eks-cluster-overview" src="https://github.com/user-attachments/assets/d67a617b-fdf4-4d55-b337-d925898a7784" />


Workloads before the load test: everything running 

<img width="1607" height="673" alt="02a-workloads-before-load-test" src="https://github.com/user-attachments/assets/44803fff-b248-4ce0-b9aa-758b41eac0a6" />


During the load test: the WordPress Deployment mid-scale-out (pending pods) 

<img width="1599" height="690" alt="02b-workloads-during-autoscaling" src="https://github.com/user-attachments/assets/5b870ce0-3492-4a3c-98e8-488827e4bccf" />


After the load test: Siege pod completed, all Deployments healthy 

<img width="1603" height="677" alt="02c-workloads-after-load-test" src="https://github.com/user-attachments/assets/ebe9bd32-144c-44f3-96ea-73a1a577b37d" />

Infrastructure as code: both eksctl CloudFormation stacks 

<img width="1901" height="329" alt="05-cloudformation-stacks" src="https://github.com/user-attachments/assets/b2cc516c-4537-4b5c-a03c-48b1f10d417d" />


Security decisions
Used a dedicated IAM user instead of root, and deleted its access key after the project
Worker nodes in private subnets with no public IPs; SSH to nodes disabled
Load balancer restricted to one IP via loadBalancerSourceRanges
Database only reachable inside the cluster (ClusterIP Service)
Passwords generated randomly and kept in a Kubernetes Secret, never in files or this repo
Pinned the MySQL image to an exact version (mysql:8.4)

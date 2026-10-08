WordPress on Amazon EKS: Helm, Autoscaling & Load Testing

Deployed a containerized WordPress + MySQL application to Amazon EKS with Helm, configured Horizontal Pod Autoscaling, and load-tested it until it broke, then traced the failure to its root cause.

<img width="3510" height="2218" alt="WordPress on Amazon EKS" src="https://github.com/user-attachments/assets/fa7169f1-3b94-4bec-8949-2b061758cb53" />


## Overview

Deployed WordPress on Amazon EKS with three worker nodes in private subnets, a custom Helm chart, an internal MySQL backend, and CPU-based horizontal pod autoscaling.

The implementation covered infrastructure provisioning, application configuration, restricted external access, and a five-minute load test with 100 concurrent users. Troubleshooting exposed gaps in the original lab guide and demonstrated why successful autoscaling does not necessarily mean a healthy application.

**Key result:** WordPress scaled from **1 to 3 replicas**, but the test recorded only **34.18% availability**. Pod diagnostics confirmed an `OOMKilled` termination, identifying memory pressure as a failure mode that CPU-based scaling alone did not resolve.

### Highlights

- Provisioned an EKS cluster using `eksctl` and CloudFormation, with managed worker nodes in private subnets.
- Packaged WordPress with Helm and added the missing MySQL backend.
- Supplied database credentials through Kubernetes Secrets rather than hard-coding them in manifests.
- Restricted the public load balancer to a single allow-listed IP address.
- Configured a Horizontal Pod Autoscaler (HPA) and observed scale-out under load.
- Investigated failed requests using pod termination states, events, and health probe failures.

## Technology Stack

| Area | Technologies |
| --- | --- |
| AWS infrastructure | Amazon EKS, EC2 (`t3.small` managed node group), VPC, NAT gateway, Classic Load Balancer, CloudFormation, IAM |
| Kubernetes | Kubernetes 1.34, Deployments, Services, Secrets, Horizontal Pod Autoscaler, metrics-server |
| Application packaging | Helm 4 |
| Application and database | WordPress, MySQL 8.4 |
| Load testing | Siege |
| Tooling | AWS CLI, `eksctl`, `kubectl`, PowerShell on Windows |

## Architecture

Incoming requests reach the public load balancer, which forwards traffic to WordPress pods on private worker nodes. WordPress connects to MySQL through an internal ClusterIP Service. The NAT gateway provides outbound connectivity for private nodes, while metrics-server supplies CPU metrics to the HPA.

| Layer | Component | Purpose |
| --- | --- | --- |
| Provisioning | `eksctl` → CloudFormation | Creates the cluster infrastructure and managed node group through separate stacks |
| Control plane | Amazon EKS | AWS-managed Kubernetes API and scheduling |
| Public entry point | Classic Load Balancer | Exposes WordPress over HTTP, with access restricted to one IP (`/32`) |
| Outbound connectivity | NAT gateway | Allows private nodes to reach the internet, including container registries |
| Compute | 3 × `t3.small` worker nodes | Runs application workloads without public node IP addresses |
| Web application | WordPress Deployment | Helm-managed workload; HPA configured for 1–5 replicas |
| Database | MySQL Deployment and ClusterIP Service | Provides database access within the cluster |
| Autoscaling | metrics-server and HPA | Uses pod CPU utilization to adjust the WordPress replica count |

## Implementation

The snippets below document the key settings and commands. They are excerpts rather than a complete deployable chart; custom values must be connected to the chart templates.

### 1. Configure tooling and AWS access

Installed `eksctl` and Helm using `winget`, then configured the AWS CLI with a dedicated IAM user instead of the root account.

### 2. Provision the EKS cluster

```powershell
eksctl create cluster --name enterprise-cluster --region us-east-1 `
  --nodegroup-name standard-workers --node-type t3.small `
  --nodes 3 --nodes-min 3 --nodes-max 6 --managed --node-private-networking
```

The `--node-private-networking` option places worker nodes in private subnets without public IP addresses. The node-group bounds are separate from the HPA's pod replica bounds.

### 3. Package WordPress with Helm

Generated the chart:

```powershell
helm create my-microservice
```

Updated `values.yaml` with the application image, restricted load balancer access, resource settings, a liveness probe, and database configuration:

```yaml
image:
  repository: wordpress
  tag: "latest"

service:
  type: LoadBalancer
  port: 80
  loadBalancerSourceRanges:
    - "YOUR-IP/32"

resources:
  requests:
    cpu: 300m
    memory: 256Mi
  limits:
    cpu: 500m
    memory: 512Mi

livenessProbe:
  httpGet:
    path: /
    port: http
  initialDelaySeconds: 30
  timeoutSeconds: 5

env:
  - name: WORDPRESS_DB_HOST
    value: mysql
  - name: WORDPRESS_DB_NAME
    value: wordpress_db
  - name: WORDPRESS_DB_USER
    value: wordpress_user
  - name: WORDPRESS_DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: mysql-secret
        key: user-password
```

Updated `deployment.yaml` and `service.yaml` so the custom environment variables and `loadBalancerSourceRanges` values were rendered into the Kubernetes manifests.

The database name and user must match the MySQL configuration below.

### 4. Configure MySQL and database credentials

Generated passwords in memory and created a Kubernetes Secret. Password values were not hard-coded in the manifests.

```powershell
$rootPw = -join ((48..57) + (65..90) + (97..122) | Get-Random -Count 24 | ForEach-Object { [char]$_ })
$userPw = -join ((48..57) + (65..90) + (97..122) | Get-Random -Count 24 | ForEach-Object { [char]$_ })

kubectl create secret generic mysql-secret `
  --from-literal=root-password=$rootPw `
  --from-literal=user-password=$userPw
```

MySQL runs in a separate Deployment with an internal ClusterIP Service named `mysql`. The container configuration includes the database, application user, and Secret references:

```yaml
containers:
  - name: mysql
    image: mysql:8.4
    env:
      - name: MYSQL_DATABASE
        value: wordpress_db
      - name: MYSQL_USER
        value: wordpress_user
      - name: MYSQL_ROOT_PASSWORD
        valueFrom:
          secretKeyRef:
            name: mysql-secret
            key: root-password
      - name: MYSQL_PASSWORD
        valueFrom:
          secretKeyRef:
            name: mysql-secret
            key: user-password
```

### 5. Deploy and verify

```powershell
helm install my-microservice ./my-microservice
kubectl get pods
kubectl get svc my-microservice
```

### 6. Configure horizontal pod autoscaling

With metrics-server available, configured the HPA to target the actual WordPress Deployment, `my-microservice`, rather than the incorrect `wordpress` name in the original guide.

```yaml
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: wordpress-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-microservice
  minReplicas: 1
  maxReplicas: 5
  targetCPUUtilizationPercentage: 50
```

### 7. Run the load test

Ran Siege inside the cluster against the WordPress Service, avoiding the public load balancer's single-IP access restriction:

```powershell
kubectl run siege --image=jstarcher/siege --restart=Never --command -- `
  /bin/sh -c "siege -c 100 -t 5m http://my-microservice/"
```

## Load Test Results

**Test conditions:** five minutes, 100 concurrent users, with requests sent from inside the cluster.

### Observed HPA behavior

CPU utilization below is relative to the configured CPU request, with an HPA target of 50%. Times are approximate observations.

| Time | CPU utilization | Replicas | Observation |
| --- | --- | --- | --- |
| Before test | 5–6% | 1 | Idle baseline |
| +15 seconds | 55% | 1 | No scaling observed near the tolerance boundary |
| +60 seconds | 111% | 1 | CPU utilization exceeded the target |
| +75 seconds | 14% | 3 | Scale-out observed; CPU utilization subsequently fell |

At 111% utilization, the simplified scaling calculation gives `ceil(1 × 111 / 50) = 3` replicas, consistent with the observed scale-out.

### Siege results

| Metric | Result |
| --- | ---: |
| Successful transactions | 242 |
| Failed transactions | 466 |
| Availability reported by Siege | **34.18%** |
| Average response time | 7.39 seconds |
| Longest transaction | 30.90 seconds |

The HPA increased the replica count, but request success remained low. Scaling activity alone was insufficient to demonstrate application reliability under this load.

## Troubleshooting

### 1. Cluster active, but no worker nodes

| Item | Finding |
| --- | --- |
| Symptom | `eksctl` timed out after approximately 25 minutes; EKS showed an active cluster with zero nodes |
| Investigation | The node group remained in `CREATING`, then rolled back; CloudFormation events exposed the failure |
| Error | `InvalidParameterCombination: The specified instance type is not eligible for Free Tier.` |
| Resolution | Deleted the partial cluster with `eksctl` and recreated it using `t3.small` |

### 2. Autoscaling succeeded, but requests failed

After scale-out, CPU utilization fell while requests continued to time out. Inspection of the original WordPress pod showed:

```text
Last State:  Terminated
Reason:      OOMKilled
Exit Code:   137
```

Pod events also showed liveness and readiness probe timeouts, followed by `connection refused` while the container restarted.

**Confirmed finding:** a WordPress container was killed for exceeding its available memory. Its configured memory limit was `512Mi`.

**Likely contributing factor:** concurrent Apache request handling increased memory demand during the 100-user test. The evidence supports memory pressure as a failure mode, but does not establish that it caused every failed transaction.

## Lessons Learned

- **Choose scaling signals that match the workload.** CPU-based autoscaling did not directly respond to the observed memory pressure.
- **Size resources from load-test evidence.** Memory limits and application concurrency settings need to be evaluated together.
- **Tune health probes under load.** Probe timeouts can cause additional restarts when an application is busy or recovering.
- **Verify chart templates and resource names.** Custom values only take effect when templates render them, and an HPA must target the correct Deployment.
- **Measure user-facing results.** Replica count, transaction success, response time, and pod health all contribute to assessing reliability.

### Next Steps

Tune Apache concurrency, including `MaxRequestWorkers`, review memory sizing and probe settings, and repeat the same load test.

Compare request success, latency, and restart counts before claiming an improvement. These changes have not yet been validated by a repeat test.


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

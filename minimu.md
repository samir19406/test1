```mermaid
flowchart TB

User[Users / QA Testers]

subgraph AWS_Cloud[AWS Cloud - Minimum for Testing]

    Route53["Route 53 - DNS
    ———
    1 Hosted Zone"]
    ACM["ACM - SSL/TLS cert
    ———
    1 Certificate"]

    subgraph VPC["VPC (10.0.0.0/16)"]

        subgraph Public_Subnets["Public Subnets (2 AZs min)"]
            ALB["ALB - Load Balancer
            ———
            Listener: 443 HTTPS"]
            NAT["NAT Gateway
            ———
            1 NAT (single AZ)"]
        end

        subgraph Private_Subnets["Private Subnets (2 AZs min)"]

            subgraph ECS_Cluster[ECS Cluster EC2]
                TG["Target Group
                ———
                Health check: /health"]
                subgraph ASG[Auto Scaling Group]
                    EC2_1["EC2 Instance x1
                    ———
                    t3.medium (2 vCPU / 4GB)
                    30GB gp3 EBS
                    Amazon Linux 2023 ECS AMI"]
                end
            end

            subgraph Aurora_Cluster["Aurora Serverless v2 PostgreSQL"]
                MasterDB["(Master DB)
                ———
                Min 0.5 ACU / Max 2 ACU
                PostgreSQL 15+
                Single AZ (no replica)
                20GB storage"]
            end

            ECR["ECR
            ———
            1 Repository
            Lifecycle: keep last 5 images"]

        end

    end

    S3["S3 - File storage
    ———
    1 Bucket
    Standard tier"]
    SES["SES - Email
    ———
    Sandbox mode (verified emails only)"]
    SQS["SQS - Queue
    ———
    1 Standard Queue"]

    subgraph CICD[CI/CD]
        GitHub["GitHub Actions
        ———
        1 Workflow (build + deploy)"]
    end

end

%% Request Flow
User -- "1 DNS lookup" --> Route53
Route53 -- "2 Resolves" --> ALB
ACM -. Certs .-> ALB
ALB -- "3 Routes" --> TG
TG -- "Routes to task" --> ASG
ASG -- Outbound via --> NAT

%% App to Data Flow
ASG -- "4 Reads/Writes" --> Aurora_Cluster
ASG -- "5 Stores files" --> S3
ASG -- "6 Sends emails" --> SES
ASG -- "7 Queues tasks" --> SQS

%% CI/CD Flow
GitHub -- "1 Pushes image" --> ECR
GitHub -- "2 Updates service" --> ECS_Cluster
ECS_Cluster -- "3 Pulls image" --> ECR
```

---

## Minimum Testing Configuration

| Service | Config | Details |
|---------|--------|---------|
| VPC | 10.0.0.0/16 | 2 public subnets + 2 private subnets across 2 AZs |
| NAT Gateway | 1x single AZ | Saves cost vs HA (1 per AZ) |
| ALB | 1x Application LB | HTTPS 443 listener, ACM cert attached |
| EC2 (ECS) | t3.medium | 2 vCPU, 4GB RAM, 30GB gp3 EBS, Amazon Linux 2023 ECS-optimized AMI |
| ECS Task | 1 task | CPU: 1024 (1 vCPU), Memory: 2048 MB |
| Aurora Serverless v2 | 0.5 – 2 ACU | PostgreSQL 15+, single AZ, no read replica, ~20GB storage |
| ECR | 1 repository | Lifecycle policy: retain last 5 images |
| S3 | 1 bucket | Standard storage class, versioning optional |
| SES | Sandbox mode | Only verified sender/receiver emails, no production approval needed |
| SQS | 1 standard queue | Default settings, 4-day retention |
| Route 53 | 1 hosted zone | 1 A record (alias to ALB) |
| ACM | 1 certificate | DNS validated, auto-renew |
| Security Groups | 3 minimum | ALB (inbound 443), EC2 (inbound from ALB only), Aurora (inbound 5432 from EC2 only) |
| IAM | ECS Task Role + Execution Role | S3, SES, SQS, ECR, CloudWatch Logs permissions |
| GitHub Actions | 1 workflow | Build image → push to ECR → update ECS service |

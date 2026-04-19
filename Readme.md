# Complete DevOps & Architecture Diagram

This document contains a comprehensive visual overview of the entire deployment pipeline and the final AWS ECS architecture we have built together for the Global PRS API.

> [!NOTE]
> All resource names, account IDs, and specific naming conventions have been anonymized in this repository for security and privacy reasons while maintaining the functional architecture.

## Deployment Pipeline Architecture

```mermaid
graph TD
    %% Define Styles
    classDef github fill:#0d1117,stroke:#30363d,stroke-width:2px,color:#fff
    classDef aws fill:#f4931a,stroke:#d17c0f,stroke-width:2px,color:#fff
    classDef ec2 fill:#e67e22,stroke:#d35400,stroke-width:2px,color:#fff
    
    %% Entities
    DEV["Developer (Writes App Code)"]
    DEVOPS["DevOps (Writes CDK Code)"]
    
    %% Github Systems
    subgraph GITHUB [GitHub DevOps Repository]
        APP["Application Source Code"]
        CDK["Infra Source Code (CDK)"]
        
        subgraph ACTIONS [GitHub Actions CI-CD]
            CI["continuous-integration.yml (Build & Push Docker)"]
            CD["continuous-deployment.yml (Update ECS Service)"]
            CDK_DEPLOY["cdk-deploy.yml (Deploy Infrastructure)"]
        end
    end
    
    %% AWS Systems
    subgraph AWS_CLOUD [AWS Cloud ap-south-1]
        OIDC["IAM OIDC Auth Role"]
        
        ECR[("Elastic Container Registry (ECR)")]
        SECRETS[("AWS Secrets Manager<br/>(prod-vihanga-secrets)")]
        CFN["AWS CloudFormation<br/>(GlobalApiProdCdkStack)"]
        
        subgraph VPC [Target VPC]
            ALB["Application Load Balancer (ALB)"]
            
            subgraph ASG [Auto Scaling Group]
                EC2["1x c6a.xlarge EC2 Instance"]
                
                subgraph ECS [ECS Cluster]
                    T1["Task 1 (App Container)"]
                    T2["Task 2 (App Container)"]
                    T3["Task 3 (App Container)"]
                    T4["Task 4 (App Container)"]
                end
            end
        end
    end
    
    %% Connections - App Pipeline
    DEV -->|"git push (App)"| APP
    APP --> CI
    CI -->|"Secure Login"| OIDC
    CI -->|"docker push"| ECR
    CI -->|"Trigger CD"| CD
    
    %% Connections - CDK Pipeline
    DEVOPS -->|"git push (Infra)"| CDK
    CDK --> CDK_DEPLOY
    CDK_DEPLOY -->|"Secure Login"| OIDC
    CDK_DEPLOY -->|"cdk deploy"| CFN
    CFN -->|"Update Infrastructure"| VPC
    
    %% ECS Deployment Update
    CD -->|"Secure Login"| OIDC
    CD -->|"ecs:UpdateService"| ECS
    
    %% Internal AWS Flow
    ALB -->|"Route Web Traffic"| ECS
    ECS -->|"Pulls new Image"| ECR
    ECS -.->|"Fetch Passwords"| SECRETS
    EC2 -->|"Hosts"| ECS
```

## How It Works (The 2 Pipelines)

### 1. The Infrastructure Flow (Right Side Flow)
1. You make a change to the CDK code on your laptop (e.g., resizing to a larger EC2 instance).
2. You run `git push` to upload the code to GitHub.
3. GitHub Actions triggers [`cdk-deploy.yml`](https://github.com/KrishnaaCloud/aws-ecs-devops-cdk-stack/blob/39f87d8eadca44ddd489b7aebeb7c5ffd2041657/Workflows/cdk-deploy.yml)
4. It secretly logs into AWS without passwords using the `OIDC` trust relationship.
5. It runs `cdk deploy` safely on AWS CloudFormation to update your subnets, ASG, Load Balancer, or Security Groups in the background.

### 2. The Application Flow (Left Side Flow)
1. The developer writes code for the actual Python API and commits it.
2. GitHub Actions triggers `continuous-integration.yml` to build the Docker Image.
3. The image is saved in AWS ECR.
4. The pipeline triggers `continuous-deployment.yml`, which talks strictly to the ECS Service.
5. ECS gracefully restarts the 4 existing tasks using the Zero-Downtime strategy to pull down the newly updated image.
6. When the new tasks start up, they dynamically pull the database passwords seamlessly out of AWS Secrets Manager using the `ecs-user` IAM Access Keys we secretly injected into the Task Definition environment!

---
title: Architecture Options
slug: architecture-options
description: AWS offers three primary hosting models for REST APIs, each with distinct tradeoffs in development speed, operational overhead, cost structure, and…
tags: ["aws","architecture","serverless","containers","ec2","ai-generated"]
publishedAt: 1784708152178
updatedAt: 1784708152178
ogImage: https://raw.githubusercontent.com/aadu999/xnote-blog/main/og/architecture-options.png
---

# Architecture Options

AWS offers three primary hosting models for REST APIs, each with distinct tradeoffs in development speed, operational overhead, cost structure, and control level. This section provides a detailed comparison to help you choose the right fit.

## The Three Paths

## 1. Serverless: API Gateway + Lambda

**Best for**: Rapid iteration, unpredictable traffic patterns, teams that want zero server management.

You write Python functions (or Node.js, Java, etc.) that respond to HTTP events from API Gateway. AWS handles provisioning, scaling, patching, and availability. You pay per invocation (ms of compute time).

```python
# aws_lambda_function.py
import json
import os
from typing import Dict, Any
from fastapi import FastAPI
from mangum import Mangum  # ASGI adapter for Lambda

app = FastAPI()

@app.get("/api/v1/users/{user_id}")
def get_user(user_id: str) -> Dict[str, Any]:
    """Fetch user by ID from DynamoDB."""
    import boto3
    dynamodb = boto3.resource('dynamodb')
    table = dynamodb.Table(os.environ['USERS_TABLE'])
    
    response = table.get_item(Key={'user_id': user_id})
    item = response.get('Item')
    
    if not item:
        raise HTTPException(status_code=404, detail="User not found")
    
    return {"user_id": item['user_id'], "name": item['name'], "email": item['email']}

@app.post("/api/v1/users", status_code=201)
def create_user(body: UserCreateSchema) -> Dict[str, Any]:
    """Create a new user."""
    import boto3
    from uuid import uuid4
    
    dynamodb = boto3.resource('dynamodb')
    table = dynamodb.Table(os.environ['USERS_TABLE'])
    
    user_id = str(uuid4())
    table.put_item(Item={
        'user_id': user_id,
        'name': body.name,
        'email': body.email,
        'created_at': datetime.utcnow().isoformat()
    })
    
    return {"user_id": user_id, "status": "created"}

# Lambda entry point
handler = Mangum(app)
```

**Mermaid Architecture Diagram — Serverless**:

```mermaid
graph TB
    subgraph Client
        A[Browser / Mobile App]
    end
    
    subgraph AWS_Shared_Tier
        B[Route 53 DNS]
        C[CloudFront CDN]
    end
    
    subgraph Security_Layer
        D[WAF Web Application Firewall]
        E[API Gateway REST/HTTP API]
        F[Cognito Authorizer]
    end
    
    subgraph Compute_Lambda
        G1[Lambda Function Users]
        G2[Lambda Function Products]
        G3[Lambda Function Orders]
    end
    
    subgraph Data_Layer
        H1[(DynamoDB)]
        H2[(RDS Postgres)]
        H3[ElastiCache Redis]
    end
    
    A --> B --> C --> D --> E --> F
    F --> G1 & G2 & G3
    G1 --> H1 & H3
    G2 --> H1
    G3 --> H2 & H3
    
    style E fill:#f9d56e,color:#000
    style G1 fill:#ffd93b,color:#000
    style G2 fill:#ffd93b,color:#000
    style G3 fill:#ffd93b,color:#000
```

## 2. Containers: ECS Fargate / EKS

**Best for**: Teams already using Docker/Kubernetes, portable workloads, need for longer-running processes or warm connections.

Package your API into a Docker container, push to Amazon ECR (Elastic Container Registry), and run it on ECS Fargate (serverless containers) or EKS (Kubernetes). You get full OS-level control while AWS manages the infrastructure.

```python
# app.py — FastAPI application for container deployment
from fastapi import FastAPI, HTTPException, Depends
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker, Session
from pydantic import BaseModel
import os

app = FastAPI(title="User API", version="2.0")

# Database setup with connection pooling
DATABASE_URL = os.getenv("DATABASE_URL", "postgresql://user:pass@localhost:5432/api_db")
engine = create_engine(
    DATABASE_URL,
    pool_size=10,          # Concurrent connections
    max_overflow=20,       # Extra under load
    pool_pre_ping=True     # Health-check idle connections
)
SessionLocal = sessionmaker(bind=engine)

def get_db() -> Session:
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()

class UserCreate(BaseModel):
    name: str
    email: str

class UserResponse(BaseModel):
    user_id: str
    name: str
    email: str
    created_at: str

@app.post("/api/v1/users", response_model=UserResponse, status_code=201)
def create_user(user: UserCreate, db: Session = Depends(get_db)):
    from models import User
    new_user = User(name=user.name, email=user.email)
    db.add(new_user)
    db.commit()
    db.refresh(new_user)
    return new_user

@app.get("/api/v1/users/{user_id}", response_model=UserResponse)
def get_user(user_id: str, db: Session = Depends(get_db)):
    from models import User
    user = db.query(User).filter(User.user_id == user_id).first()
    if not user:
        raise HTTPException(status_code=404, detail="User not found")
    return user

@app.get("/health", status_code=200)
def health_check():
    """ALB health check endpoint."""
    return {"status": "healthy", "version": "2.0"}
```

**Mermaid Architecture Diagram — ECS Fargate**:

```mermaid
graph TB
    subgraph Client
        A[Browser / Mobile App]
    end
    
    subgraph Network
        B[Application Load Balancer]
        C[Target Group]
    end
    
    subgraph ECS_Cluster
        D1[Task Definition: User API]
        D2[Task Definition: Order API]
        
        subgraph Fargate_Instances
            E1[f5g1: API Container 1]
            E2[f5g2: API Container 2]
            E3[f5g3: API Container 3]
        end
    end
    
    subgraph Registry_and_Data
        F[(Amazon ECR)]
        H1[(RDS Aurora)]
        H2[ElastiCache Redis]
        H3[CloudWatch Logs]
    end
    
    A --> B --> C --> E1 & E2 & E3
    D1 --> E1
    D2 --> E2
    F -.->|Pull Image| E1 & E2 & E3
    E1 & E2 & E3 --> H1 & H2
    E1 & E2 & E3 --> H3
    
    style B fill:#e74c3c,color:#fff
    style E1 fill:#6c5ce7,color:#fff
    style E2 fill:#6c5ce7,color:#fff
    style E3 fill:#6c5ce7,color:#fff
```

## 3. VMs: EC2 behind API Gateway or ALB

**Best for**: Legacy applications, GPU/ specialty instances, maximum OS control, predictable high-throughput workloads where Reserved Instances yield lowest cost.

You provision EC2 instances, install your runtime and dependencies manually (or via Packer/Chef/Ansible), and place them behind an Application Load Balancer or API Gateway's custom authorizer with a private integration.

```python
# gunicorn_config.py — WSGI/ASGI server configuration for EC2 deployment
import multiprocessing

# Server socket settings
bind = "0.0.0.0:8000"
workers = multiprocessing.cpu_count() * 2 + 1  # Optimal worker count
worker_class = "uvicorn.workers.UvicornWorker"  # For ASGI (FastAPI)
timeout = 120  # Seconds before killing idle workers

# Logging
accesslog = "-"  # stdout
errorlog = "-"   # stdout
loglevel = "info"

# Process naming
proc_name = "scalable-api"

# Graceful restart
graceful_timeout = 30
```

```bash
# systemd service file: /etc/systemd/system/api.service
[Unit]
Description=Scalable REST API Service
After=network.target postgresql.service

[Service]
Type=notify
User=apiuser
Group=apiuser
WorkingDirectory=/opt/api/
Environment="PATH=/opt/api/venv/bin"
ExecStart=/opt/api/venv/bin/gunicorn app:app -c gunicorn_config.py
ExecReload=/bin/kill -s HUP $MAINPID
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

**Mermaid Architecture Diagram — EC2**:

```mermaid
graph TB
    subgraph Client
        A[Browser / Mobile App]
    end
    
    subgraph VPC_Public_Subnet
        B[Internet Gateway]
        C[NAT Gateway]
        D[Application Load Balancer]
    end
    
    subgraph VPC_Private_Subnet_EZ_1
        E1[EC2 Instance m6g.large 1]
        E2[EC2 Instance m6g.large 2]
    end
    
    subgraph VPC_Private_Subnet_EZ_2
        F1[EC2 Instance m6g.large 3]
        F2[EC2 Instance m6g.large 4]
    end
    
    subgraph Data_Tier
        G1[(RDS Multi-AZ Primary)]
        G2[(RDS Multi-AZ Replica)]
        G3[Auto Scaling Group]
    end
    
    A --> B --> D
    D --> E1 & E2 & F1 & F2
    E1 & E2 & F1 & F2 --> G1
    G1 <--> G2
    C -.->|Outbound Internet for patches| E1 & E2
    G3 -->|Manages scaling of| D
    
    style D fill:#e74c3c,color:#fff
    style E1 fill:#0984e3,color:#fff
    style E2 fill:#0984e3,color:#fff
    style F1 fill:#0984e3,color:#fff
    style F2 fill:#0984e3,color:#fff
```

## Decision Matrix

| Factor | Serverless Lambda | ECS / EKS | EC2 |
|---|---|---|---|
| **Cold Start** | 10-500ms (mitigated with provisioned conc.) | None (always warm) | None |
| **Max Request Duration** | 15 minutes | Unlimited | Unlimited |
| **Scaling Speed** | Sub-second to thousands of instances | Seconds to minutes (depends on config) | Minutes (launch delay) |
| **Per-Request Cost at Low Volume** | Very low (pay per ms) | Higher (container always running) | Highest (instance always on) |
| **Per-Request Cost at High Volume** | Can exceed EC2 beyond sustained load | Competitive | Cheapest with Reserved Instances |
| **Operational Overhead** | Minimal — AWS handles everything | Moderate — manage tasks/manifests | High — patch OS, manage servers |
| **Portability** | Vendor-locked to AWS Lambda format | High — standard Docker/K8s manifests | Low — tied to EC2 AMIs |
| **Stateful Connections** | Poor (request/stateless model) | Good (container stays alive) | Best (full control) |
| **Security Isolation** | Per-invocation isolation | Per-container isolation | Per-VM isolation |
| **Custom AMI/Runtime Needs** | Limited (Lambda runtime constraints) | Flexibility within container | Full OS control |

## Capacity Planning Heuristics

```python
# capacity_estimator.py — Rough cost model for choosing between options
import math

def estimate_serverless_cost(rps: float, avg_duration_ms: float, memory_mb: int = 512) -> dict:
    """Estimate monthly AWS Lambda cost."""
    # $0.20 per 1M requests + $0.0000166667 per GB-second
    seconds_per_month = 30 * 24 * 3600
    total_requests = rps * seconds_per_month
    
    request_cost = (total_requests / 1_000_000) * 0.20
    compute_gb_seconds = total_requests * (avg_duration_ms / 1000) * (memory_mb / 1024)
    compute_cost = compute_gb_seconds * 0.0000166667
    
    return {
        "approach": "Lambda Serverless",
        "monthly_request_cost_usd": round(request_cost, 2),
        "monthly_compute_cost_usd": round(compute_cost, 2),
        "total_monthly_usd": round(request_cost + compute_cost, 2),
        "estimated_concurrent_executions": math.ceil(rps * avg_duration_ms / 1000)
    }

def estimate_ec2_cost(rps: float, requests_per_instance: float = 500, instance_type: str = "t3.medium") -> dict:
    """Estimate monthly EC2 cost (on-demand hourly pricing)."""
    # Instance hourly prices (approx): t3.medium=$0.0416, m6g.large=$0.096
    hourly_prices = {"t3.medium": 0.0416, "m6g.large": 0.096, "c5n.xlarge": 0.17}
    hours_per_month = 730
    
    instances_needed = math.ceil(rps / requests_per_instance)
    hourly_price = hourly_prices.get(instance_type, 0.0416)
    
    on_demand_cost = instances_needed * hourly_price * hours_per_month
    reserved_1yr_discount = 0.35 * on_demand_cost  # ~35% savings with 1-yr RI
    
    return {
        "approach": "EC2",
        "instances_needed": instances_needed,
        "monthly_on_demand_usd": round(on_demand_cost, 2),
        "monthly_1yr_reserved_usd": round(on_demand_cost - reserved_1yr_discount, 2)
    }

def estimate_ecs_cost(rps: float, avg_cpu_percent_per_request: float = 0.5, container_vcpu: int = 1) -> dict:
    """Estimate monthly ECS Fargate cost."""
    # Fargate: $0.04048 per vCPU-second + $0.004445 per GB-second
    seconds_per_month = 30 * 24 * 3600
    total_cpu_seconds = rps * seconds_per_month * (avg_cpu_percent_per_request / 100)
    
    cpu_cost = total_cpu_seconds * 0.04048 / 1000  # normalized
    memory_gb_seconds = total_cpu_seconds  # simplified
    mem_cost = memory_gb_seconds * 0.004445 / 1000
    
    tasks_needed = math.ceil(rps / (container_vcpu * 100))  # rough throughput estimate
    
    return {
        "approach": "ECS Fargate",
        "tasks_needed": tasks_needed,
        "monthly_compute_usd": round(cpu_cost + mem_cost, 2)
    }

# Example usage:
if __name__ == "__main__":
    rps = 100  # 100 requests per second
    
    print("=== Cost Estimates at", rps, "RPS ===\n")
    for result in [
        estimate_serverless_cost(rps, avg_duration_ms=80),
        estimate_ec2_cost(rps),
        estimate_ecs_cost(rps)
    ]:
        print(f"{result['approach']}: ${result.get('total_monthly_usd', result.get('monthly_on_demand_usd', 0))}/mo")
```

## Recommendation Flow

```mermaid
flowchart TD
    A[Start: Need REST API on AWS] --> B{Team size?}
    B -->|1-5 engineers| C{Traffic pattern?}
    B -->|6+ with DevOps| D{Need K8s / Docker ecosystem?}
    
    C -->|Spike-prone / unpredictable| E[Serverless Lambda]
    C -->|Steady / predictable| F{Expected volume?}
    F -->|< 100 RPS avg| E
    F -->|100-1000 RPS avg| G[ECS Fargate]
    F -->|1000+ RPS avg| H[EC2 Reserved]
    
    D -->|Yes| I[EKS Kubernetes]
    D -->|No| J{Full OS control needed?}
    J -->|Yes| K[EC2 + ALB]
    J -->|No| G
    
    E --> L[Deploy]
    G --> L
    H --> L
    I --> L
    K --> L
```
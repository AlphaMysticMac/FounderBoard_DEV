# FastAPI Deployment on AWS Fargate with Load Balancer
### Region: ap-south-2 (Hyderabad) | April 2026

---

## Prerequisites

- AWS account with admin or sufficient IAM access
- Docker Desktop installed locally (with BuildKit support)
- AWS CLI v2 installed and configured (`aws configure`)
- Your FastAPI project ready locally

---

## PART 1 — Prepare Your FastAPI Project Locally

### Step 1.1 — Confirm your project structure

Your project should look like this at minimum:

```
my-fastapi-app/
├── main.py
├── requirements.txt
└── Dockerfile
```

### Step 1.2 — Ensure `uvicorn` is in requirements.txt

```
fastapi
uvicorn[standard]
```

Add any other dependencies your app needs.

### Step 1.3 — Create the Dockerfile

Create a file named `Dockerfile` (no extension) in your project root:

```dockerfile
# Use official Python slim image — AMD64 platform explicitly
FROM --platform=linux/amd64 python:3.11-slim

# Set working directory
WORKDIR /app

# Copy and install dependencies first (layer caching)
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application code
COPY . .

# Expose port 7050
EXPOSE 7050

# Start the app with uvicorn on port 7050
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "7050"]
```

> **Note:** Replace `main:app` if your FastAPI instance is named differently (e.g., `app.main:app`).

---

## PART 2 — Build and Push Docker Image to Amazon ECR

### Step 2.1 — Create ECR Repository (AWS Console)

1. Go to **AWS Console** → search **ECR** → **Elastic Container Registry**
2. Confirm you are in region **Asia Pacific (Hyderabad) ap-south-2** (top-right dropdown)
3. Click **Create repository**
4. Set:
   - **Visibility**: Private
   - **Repository name**: `my-fastapi-app` (or your preferred name)
   - **Tag immutability**: Disabled (for now)
   - **Encryption**: AES-256 (default)
5. Click **Create repository**
6. Open the repository and copy the **URI** — it looks like:
   ```
   123456789012.dkr.ecr.ap-south-2.amazonaws.com/my-fastapi-app
   ```

### Step 2.2 — Authenticate Docker to ECR (Local Terminal)

Run this command (replace `123456789012` with your AWS account ID):

```bash
aws ecr get-login-password --region ap-south-2 | \
  docker login --username AWS --password-stdin \
  123456789012.dkr.ecr.ap-south-2.amazonaws.com
```

You should see: `Login Succeeded`

### Step 2.3 — Build the Docker Image for AMD64

```bash
docker buildx build \
  --platform linux/amd64 \
  -t my-fastapi-app:latest \
  .
```

> If you are on an Apple Silicon Mac (M1/M2/M3), the `--platform linux/amd64` flag is essential to build the correct architecture for Fargate.

### Step 2.4 — Tag the Image

```bash
docker tag my-fastapi-app:latest \
  123456789012.dkr.ecr.ap-south-2.amazonaws.com/my-fastapi-app:latest
```

### Step 2.5 — Push the Image to ECR

```bash
docker push 123456789012.dkr.ecr.ap-south-2.amazonaws.com/my-fastapi-app:latest
```

Verify: Go to ECR Console → your repository → you should see the `latest` tag with a recent push time.

---

## PART 3 — Networking: VPC and Security Groups

> If you want to use the default VPC, you can skip Step 3.1 and use the existing default VPC. A new VPC is recommended for production.

### Step 3.1 — Create a VPC (AWS Console)

1. Go to **VPC** service → **Your VPCs** → **Create VPC**
2. Set:
   - **Name**: `fastapi-vpc`
   - **IPv4 CIDR**: `10.0.0.0/16`
   - **Tenancy**: Default
3. Click **Create VPC**

### Step 3.2 — Create Subnets

You need at least **2 public subnets** in different Availability Zones (required by ALB).

1. Go to **Subnets** → **Create subnet**
2. Select your VPC (`fastapi-vpc`)
3. Create **Subnet 1**:
   - **Name**: `fastapi-public-1`
   - **AZ**: `ap-south-2a`
   - **CIDR**: `10.0.1.0/24`
4. Create **Subnet 2**:
   - **Name**: `fastapi-public-2`
   - **AZ**: `ap-south-2b`
   - **CIDR**: `10.0.2.0/24`

### Step 3.3 — Create and Attach an Internet Gateway

1. Go to **Internet Gateways** → **Create internet gateway**
2. Name it `fastapi-igw` → **Create**
3. Select it → **Actions** → **Attach to VPC** → select `fastapi-vpc`

### Step 3.4 — Update Route Table for Public Subnets

1. Go to **Route Tables** → find the route table associated with your VPC
2. Select it → **Routes** tab → **Edit routes**
3. Add route:
   - **Destination**: `0.0.0.0/0`
   - **Target**: select your internet gateway `fastapi-igw`
4. Save
5. Go to **Subnet associations** tab → **Edit subnet associations** → select both subnets → **Save**

### Step 3.5 — Enable Auto-assign Public IP on Both Subnets

1. Go to **Subnets** → select `fastapi-public-1`
2. **Actions** → **Edit subnet settings**
3. Check **Enable auto-assign public IPv4 address** → **Save**
4. Repeat for `fastapi-public-2`

### Step 3.6 — Create Security Groups

**Security Group 1: For the Load Balancer**

1. Go to **Security Groups** → **Create security group**
2. Set:
   - **Name**: `fastapi-alb-sg`
   - **VPC**: `fastapi-vpc`
3. **Inbound rules**:
   - Type: HTTP, Port: 80, Source: `0.0.0.0/0`
   - *(Optional)* Type: HTTPS, Port: 443, Source: `0.0.0.0/0`
4. **Outbound rules**: Allow all (default)
5. **Create security group**

**Security Group 2: For ECS Tasks (Fargate)**

1. Create another security group:
   - **Name**: `fastapi-ecs-sg`
   - **VPC**: `fastapi-vpc`
2. **Inbound rules**:
   - Type: Custom TCP, Port: `7050`, Source: select `fastapi-alb-sg` (the ALB security group)
3. **Outbound rules**: Allow all (default)
4. **Create security group**

---

## PART 4 — Application Load Balancer (ALB)

### Step 4.1 — Create the ALB

1. Go to **EC2** → **Load Balancers** → **Create load balancer**
2. Choose **Application Load Balancer** → **Create**
3. Configure:
   - **Name**: `fastapi-alb`
   - **Scheme**: Internet-facing
   - **IP address type**: IPv4
   - **VPC**: `fastapi-vpc`
   - **Mappings**: Select both `ap-south-2a` (fastapi-public-1) and `ap-south-2b` (fastapi-public-2)
   - **Security groups**: Remove the default, add `fastapi-alb-sg`
4. **Listeners and routing** — for now, set:
   - Protocol: HTTP, Port: 80
   - Default action: **Create target group** (click the link — opens in a new tab)

### Step 4.2 — Create a Target Group

(This opens in a new browser tab from the previous step, or go to **EC2** → **Target Groups** → **Create target group**)

1. Choose **IP addresses** as target type (required for Fargate)
2. Set:
   - **Name**: `fastapi-tg`
   - **Protocol**: HTTP
   - **Port**: `7050`
   - **VPC**: `fastapi-vpc`
3. **Health check settings**:
   - Protocol: HTTP
   - Path: `/` (or your app's health endpoint like `/health`)
   - Healthy threshold: 2
   - Interval: 30 seconds
4. Click **Next** → **Create target group** (do not register any targets manually — ECS will do this)

### Step 4.3 — Finish ALB Creation

Go back to the ALB creation tab:
- Under listener HTTP:80, set **Default action** → **Forward to** → select `fastapi-tg`
- Click **Create load balancer**

Copy the **DNS name** of the ALB (e.g., `fastapi-alb-123456.ap-south-2.elb.amazonaws.com`) — you will use this to access your app.

---

## PART 5 — IAM Role for ECS Task Execution

### Step 5.1 — Create an ECS Task Execution Role

1. Go to **IAM** → **Roles** → **Create role**
2. **Trusted entity type**: AWS service
3. **Use case**: Select **Elastic Container Service** → then choose **Elastic Container Service Task**
4. Click **Next**
5. Search and attach the policy: `AmazonECSTaskExecutionRolePolicy`
6. Click **Next**
7. **Role name**: `ecsTaskExecutionRole`
8. **Create role**

> This role allows Fargate to pull your image from ECR and write logs to CloudWatch.

---

## PART 6 — ECS Cluster and Task Definition

### Step 6.1 — Create an ECS Cluster

1. Go to **ECS** → **Clusters** → **Create cluster**
2. Set:
   - **Cluster name**: `fastapi-cluster`
   - **Infrastructure**: Select **AWS Fargate (serverless)** only
3. Click **Create**

### Step 6.2 — Create a Task Definition

1. Go to **ECS** → **Task definitions** → **Create new task definition**
2. **Task definition family name**: `fastapi-task`
3. **Launch type**: AWS Fargate
4. **OS/Architecture**: Linux/X86_64 (AMD64)
5. **Task size**:
   - CPU: `0.5 vCPU` (512)
   - Memory: `1 GB` (1024)
   *(Scale these up based on your app's needs)*
6. **Task execution role**: `ecsTaskExecutionRole`
7. Under **Container — 1**:
   - **Name**: `fastapi-container`
   - **Image URI**: paste your ECR image URI:
     ```
     123456789012.dkr.ecr.ap-south-2.amazonaws.com/my-fastapi-app:latest
     ```
   - **Container port**: `7050`
   - **Protocol**: TCP
8. **Logging**: Select **Use log collection** → choose **Amazon CloudWatch** (auto-creates a log group)
9. Click **Create**

---

## PART 7 — ECS Service (Connects Everything Together)

### Step 7.1 — Create the ECS Service

1. Go to **ECS** → **Clusters** → `fastapi-cluster` → **Services** tab → **Create**
2. Configure:
   - **Compute options**: Launch type → **FARGATE**
   - **Task definition**: `fastapi-task` → select latest revision
   - **Service name**: `fastapi-service`
   - **Desired tasks**: `1` (start with 1 to verify, scale up later)
3. **Networking**:
   - **VPC**: `fastapi-vpc`
   - **Subnets**: select both `fastapi-public-1` and `fastapi-public-2`
   - **Security group**: remove default → add `fastapi-ecs-sg`
   - **Public IP**: **Turn on** (required so Fargate can pull from ECR in public subnets)
4. **Load balancing**:
   - Select **Application Load Balancer**
   - **Load balancer**: `fastapi-alb`
   - **Listener**: `80:HTTP`
   - **Target group**: `fastapi-tg`
5. Click **Create**

---

## PART 8 — Verify the Deployment

### Step 8.1 — Watch the Task Start

1. Go to **ECS** → **Clusters** → `fastapi-cluster` → **Tasks** tab
2. You should see a task in `PROVISIONING` → `PENDING` → `RUNNING` state
3. This takes about 1-3 minutes on first launch

### Step 8.2 — Check Task Health

1. Click the running task
2. Scroll to **Containers** → look for status **RUNNING**
3. Check **Logs** tab for uvicorn startup messages like:
   ```
   INFO:     Uvicorn running on http://0.0.0.0:7050
   ```

### Step 8.3 — Check Target Group Health

1. Go to **EC2** → **Target Groups** → `fastapi-tg`
2. Click **Targets** tab
3. Wait for the target to show **healthy** (may take 1-2 minutes)
4. If it shows **unhealthy**, check the health check path in Step 4.2 — make sure your app returns HTTP 200 on `/`

### Step 8.4 — Access Your Application

Open a browser and navigate to your ALB DNS name:

```
http://fastapi-alb-123456.ap-south-2.elb.amazonaws.com/docs
```

You should see your FastAPI Swagger UI.

---

## PART 9 — Update Your App (Future Deployments)

When you change your code, repeat these steps:

```bash
# 1. Rebuild image
docker buildx build --platform linux/amd64 -t my-fastapi-app:latest .

# 2. Re-tag
docker tag my-fastapi-app:latest \
  123456789012.dkr.ecr.ap-south-2.amazonaws.com/my-fastapi-app:latest

# 3. Push
docker push 123456789012.dkr.ecr.ap-south-2.amazonaws.com/my-fastapi-app:latest
```

Then in the AWS Console:
1. Go to **ECS** → **Clusters** → `fastapi-cluster` → **Services** → `fastapi-service`
2. Click **Update service**
3. Check **Force new deployment**
4. Click **Update**

ECS will pull the new image and do a rolling update with zero downtime.

---

## Quick Reference: Resources Created

| Resource | Name | Purpose |
|----------|------|---------|
| ECR Repository | `my-fastapi-app` | Stores Docker image |
| VPC | `fastapi-vpc` | Network isolation |
| Subnets | `fastapi-public-1/2` | Two AZs for ALB |
| Internet Gateway | `fastapi-igw` | Internet access |
| Security Group | `fastapi-alb-sg` | ALB inbound HTTP |
| Security Group | `fastapi-ecs-sg` | ECS port 7050 from ALB |
| ALB | `fastapi-alb` | Public-facing load balancer |
| Target Group | `fastapi-tg` | Routes to ECS tasks on :7050 |
| IAM Role | `ecsTaskExecutionRole` | Fargate pulls ECR image |
| ECS Cluster | `fastapi-cluster` | Fargate cluster |
| Task Definition | `fastapi-task` | Container spec (AMD64, port 7050) |
| ECS Service | `fastapi-service` | Runs and maintains tasks |

---

## Common Issues & Fixes

**Task stuck in PENDING or STOPPED**
- Check CloudWatch logs: ECS → Task → Logs tab
- Common cause: ECR pull failure — ensure Public IP is turned on for the task, or use a NAT Gateway if in private subnets

**Target group showing unhealthy**
- Your app must return HTTP 200 on the health check path (default `/`)
- Add a health endpoint if needed: `@app.get("/health") def health(): return {"status": "ok"}`

**Cannot connect to ALB DNS**
- Check ALB security group allows inbound port 80
- Check listener is forwarding to the target group
- Allow 2-3 minutes after the task reaches RUNNING state

**Wrong architecture error**
- Ensure your Dockerfile has `FROM --platform=linux/amd64`
- Ensure your build command includes `--platform linux/amd64`

# Building Enterprise Real-Time Applications with Kafka, Couchbase, and AWS
## Complete Udemy Course Structure

**Course Duration:** ~15-20 hours  
**Level:** Beginner to Intermediate  
**Prerequisites:** Basic Python knowledge, familiarity with command line

---

## Course Overview

Learn to build a production-ready, real-time order processing system from scratch. This hands-on course covers everything from writing Python microservices to deploying on AWS using Terraform.

**What You'll Build:**
- Real-time order processing pipeline
- Kafka producer and 3 consumer services
- FastAPI backend with Couchbase database
- React frontend dashboard
- Complete AWS ECS deployment with Terraform

---

## MODULE 1: Introduction & Project Overview (30 minutes)

### Video 1.1: Welcome & Course Introduction (5 min)
**Script Outline:**
- [ ] Welcome students
- [ ] What you'll learn in this course
- [ ] What we're building (show live demo: https://orders.jumptotech.net)
- [ ] Course structure overview
- [ ] Prerequisites check

**Key Points:**
- "By the end, you'll have deployed a real-time system to AWS"
- "We'll build everything step-by-step, no prior Kafka or AWS experience needed"

### Video 1.2: System Architecture Overview (10 min)
**Script Outline:**
- [ ] Show architecture diagram
- [ ] Explain data flow: Producer → Kafka → Consumers → Database → Backend → Frontend
- [ ] Explain each component's role
- [ ] Why we chose these technologies
- [ ] Real-world use cases

**Visuals:**
- Architecture diagram on screen
- Animated flow showing data movement
- Highlight each component as you explain

**Key Points:**
- "Kafka handles real-time streaming"
- "Three consumers process orders in parallel"
- "Couchbase stores the data"
- "React frontend shows live dashboard"

### Video 1.3: Setting Up Your Development Environment (15 min)
**Script Outline:**
- [ ] Install Python 3.11
- [ ] Install Docker Desktop
- [ ] Install VS Code (or preferred IDE)
- [ ] Install Git
- [ ] Clone the repository
- [ ] Verify installations

**Hands-on:**
- Show each installation step
- Test Docker: `docker --version`
- Test Python: `python --version`
- Clone repo: `git clone https://github.com/...`

---

## MODULE 2: Understanding Kafka Basics (45 minutes)

### Video 2.1: What is Kafka and Why Use It? (10 min)
**Script Outline:**
- [ ] What is Kafka? (message queue/streaming platform)
- [ ] Real-world analogy (post office, event bus)
- [ ] Key concepts: Topics, Partitions, Producers, Consumers
- [ ] Why Kafka for real-time systems?
- [ ] Kafka vs traditional databases

**Key Points:**
- "Kafka is like a post office for messages"
- "Topics are like mailboxes"
- "Producers send, Consumers receive"

### Video 2.2: Kafka Core Concepts Deep Dive (15 min)
**Script Outline:**
- [ ] Topics and Partitions (explain with diagram)
- [ ] Consumer Groups (why we need 3 groups)
- [ ] Offsets (how Kafka tracks what you've read)
- [ ] Replication and fault tolerance
- [ ] Show Confluent Cloud interface

**Visuals:**
- Diagram showing topic with partitions
- Show 3 consumer groups reading same topic
- Explain offset tracking

**Key Points:**
- "Each consumer group reads independently"
- "Offsets prevent reading the same message twice"

### Video 2.3: Setting Up Confluent Cloud (Free Tier) (20 min)
**Script Outline:**
- [ ] Create Confluent Cloud account
- [ ] Create a cluster (free tier)
- [ ] Create API key and secret
- [ ] Create "orders" topic
- [ ] Test connection with kafka-console-producer
- [ ] Save credentials securely

**Hands-on:**
- Walk through Confluent Cloud signup
- Show cluster creation
- Generate API keys
- Create topic via UI
- Test with command line tools

**Key Points:**
- "We'll use Confluent Cloud (managed Kafka)"
- "Free tier is perfect for learning"
- "Save your credentials - we'll need them later"

---

## MODULE 3: Building the Producer Service (60 minutes)

### Video 3.1: Project Structure & Producer Overview (10 min)
**Script Outline:**
- [ ] Show project folder structure
- [ ] Explain producer's role
- [ ] What data we'll generate (orders)
- [ ] Show producer.py file structure

**Visuals:**
- File tree diagram
- Highlight producer/ folder
- Show sample order JSON

### Video 3.2: Writing the Producer Code - Part 1 (15 min)
**Script Outline:**
- [ ] Create producer.py file
- [ ] Import required libraries (kafka-python, faker, json)
- [ ] Set up environment variables
- [ ] Create Kafka producer connection
- [ ] Explain SASL_SSL configuration

**Code Walkthrough:**
```python
# Show each line and explain
from kafka import KafkaProducer
import os
import json

BOOTSTRAP = os.environ.get("KAFKA_BOOTSTRAP_SERVERS")
API_KEY = os.environ.get("CONFLUENT_API_KEY")
# ... explain each part
```

**Key Points:**
- "Environment variables keep secrets safe"
- "SASL_SSL is for secure connection to Confluent"

### Video 3.3: Writing the Producer Code - Part 2 (15 min)
**Script Outline:**
- [ ] Generate fake order data
- [ ] Create order structure (order_id, amount, country, etc.)
- [ ] Send order to Kafka topic
- [ ] Handle errors
- [ ] Add logging

**Code Walkthrough:**
```python
def generate_order(order_id):
    return {
        "order_id": order_id,
        "amount": round(random.uniform(10, 500), 2),
        "country": random.choice(["US", "CA", "DE", ...]),
        # ... explain each field
    }
```

**Key Points:**
- "Faker library generates realistic data"
- "We send JSON to Kafka"

### Video 3.4: Testing the Producer Locally (20 min)
**Script Outline:**
- [ ] Set environment variables
- [ ] Run producer.py
- [ ] Check Confluent Cloud UI - see messages arriving
- [ ] Verify messages in topic
- [ ] Debug common issues

**Hands-on:**
- Run: `python producer.py`
- Show messages appearing in Confluent Cloud
- Explain what you see
- Fix any connection errors

**Key Points:**
- "Watch messages appear in real-time"
- "Each message has partition and offset"

---

## MODULE 4: Building Consumer Services (90 minutes)

### Video 4.1: Understanding Consumer Groups (10 min)
**Script Outline:**
- [ ] What is a consumer?
- [ ] Consumer groups explained
- [ ] Why 3 separate services?
- [ ] Show architecture: one topic, three groups

**Visuals:**
- Diagram showing 3 consumer groups
- Each reading from same topic independently

**Key Points:**
- "Each consumer group processes ALL messages"
- "Groups work independently"

### Video 4.2: Building the Fraud Detection Service (20 min)
**Script Outline:**
- [ ] Create fraud-service folder
- [ ] Write fraud_consumer.py
- [ ] Connect to Kafka
- [ ] Implement fraud detection logic
- [ ] Send alerts to new topic

**Code Walkthrough:**
```python
# Show fraud detection logic
if amount > 300 and country not in ["US", "CA"]:
    # Flag as suspicious
    # Send to fraud-alerts topic
```

**Key Points:**
- "Fraud service checks every order"
- "Suspicious orders go to separate topic"

### Video 4.3: Building the Payment Service (20 min)
**Script Outline:**
- [ ] Create payment-service folder
- [ ] Write payment_consumer.py
- [ ] Simulate payment processing
- [ ] Update order status
- [ ] Send to payments topic

**Code Walkthrough:**
```python
# Show payment processing
time.sleep(0.1)  # Simulate payment delay
order["status"] = "PAID"
producer.send("payments", value=order)
```

**Key Points:**
- "Payment service simulates processing"
- "Updates order status"

### Video 4.4: Building the Analytics Service (25 min)
**Script Outline:**
- [ ] Create analytics-service folder
- [ ] Write analytics_consumer.py
- [ ] Connect to Kafka
- [ ] Calculate running totals
- [ ] Connect to Couchbase (next module)
- [ ] Save orders to database

**Code Walkthrough:**
```python
# Show analytics logic
total_sales += amount
order_count += 1
# Save to Couchbase (preview)
```

**Key Points:**
- "Analytics service tracks totals"
- "Saves data for dashboard"

### Video 4.5: Testing All Consumers (15 min)
**Script Outline:**
- [ ] Run all three consumers
- [ ] Show messages being processed
- [ ] Verify each service works
- [ ] Check logs
- [ ] Debug issues

**Hands-on:**
- Run producer
- Run all 3 consumers in separate terminals
- Show logs from each
- Verify data flow

---

## MODULE 5: Setting Up Couchbase Database (60 minutes)

### Video 5.1: What is Couchbase? (10 min)
**Script Outline:**
- [ ] What is Couchbase? (NoSQL document database)
- [ ] Why Couchbase for this project?
- [ ] Key features: JSON documents, N1QL queries
- [ ] Couchbase vs MongoDB vs PostgreSQL

**Key Points:**
- "Couchbase stores JSON documents"
- "N1QL is like SQL for JSON"

### Video 5.2: Setting Up Couchbase Capella (Free Tier) (20 min)
**Script Outline:**
- [ ] Create Couchbase Capella account
- [ ] Create database cluster
- [ ] Create bucket "order_analytics"
- [ ] Create database user
- [ ] Get connection string
- [ ] Test connection

**Hands-on:**
- Walk through Capella signup
- Create cluster (free tier)
- Create bucket
- Show connection details

**Key Points:**
- "Capella is managed Couchbase"
- "Free tier is perfect for learning"

### Video 5.3: Connecting Analytics Service to Couchbase (20 min)
**Script Outline:**
- [ ] Install couchbase Python library
- [ ] Add connection code to analytics service
- [ ] Connect to Couchbase
- [ ] Save orders to database
- [ ] Handle connection errors

**Code Walkthrough:**
```python
from couchbase.cluster import Cluster
from couchbase.auth import PasswordAuthenticator

cluster = Cluster("couchbases://host", 
    ClusterOptions(PasswordAuthenticator(user, password)))
bucket = cluster.bucket("order_analytics")
collection = bucket.default_collection()
collection.upsert(doc_id, order)
```

**Key Points:**
- "couchbases:// is for SSL connection"
- "upsert creates or updates document"

### Video 5.4: Testing Database Integration (10 min)
**Script Outline:**
- [ ] Run analytics service
- [ ] Verify orders in Couchbase UI
- [ ] Query documents
- [ ] Show JSON structure

**Hands-on:**
- Run analytics consumer
- Show documents appearing in Capella UI
- Query a document
- Show JSON structure

---

## MODULE 6: Building the Backend API (90 minutes)

### Video 6.1: Introduction to FastAPI (10 min)
**Script Outline:**
- [ ] What is FastAPI?
- [ ] Why FastAPI? (fast, auto-docs, async)
- [ ] Show API documentation feature
- [ ] Basic FastAPI structure

**Key Points:**
- "FastAPI is modern Python web framework"
- "Auto-generates API documentation"

### Video 6.2: Setting Up FastAPI Project (15 min)
**Script Outline:**
- [ ] Create backend folder structure
- [ ] Create requirements.txt
- [ ] Install FastAPI and dependencies
- [ ] Create basic app.py
- [ ] Run first API server

**Code Walkthrough:**
```python
from fastapi import FastAPI
app = FastAPI()

@app.get("/")
def root():
    return {"message": "Hello World"}
```

**Hands-on:**
- Create project structure
- Install dependencies
- Run: `uvicorn app:app --reload`
- Show auto-generated docs at /docs

### Video 6.3: Connecting Backend to Couchbase (20 min)
**Script Outline:**
- [ ] Add Couchbase connection code
- [ ] Create connection function
- [ ] Handle connection errors
- [ ] Test connection

**Code Walkthrough:**
```python
# Show connection setup
cluster = Cluster(conn_str, ClusterOptions(...))
bucket = cluster.bucket(COUCHBASE_BUCKET)
```

**Key Points:**
- "Reuse same connection pattern"
- "Handle errors gracefully"

### Video 6.4: Creating the Analytics Endpoint (20 min)
**Script Outline:**
- [ ] Create /api/analytics endpoint
- [ ] Write N1QL query
- [ ] Fetch orders from Couchbase
- [ ] Return JSON response
- [ ] Test endpoint

**Code Walkthrough:**
```python
@app.get("/api/analytics")
def get_analytics():
    query = "SELECT * FROM order_analytics LIMIT 10"
    result = cluster.query(query)
    return {"orders": list(result)}
```

**Hands-on:**
- Write endpoint
- Test with curl or browser
- Show JSON response

### Video 6.5: Adding Health Check Endpoint (10 min)
**Script Outline:**
- [ ] Create /healthz endpoint
- [ ] Why health checks matter
- [ ] Test health endpoint
- [ ] Use in deployment

**Code Walkthrough:**
```python
@app.get("/healthz")
def healthz():
    return {"status": "ok"}
```

### Video 6.6: Testing the Complete Backend (15 min)
**Script Outline:**
- [ ] Run backend
- [ ] Test all endpoints
- [ ] Show API documentation
- [ ] Verify data flow: Analytics → Couchbase → Backend → API

**Hands-on:**
- Run backend
- Test /api/analytics
- Test /healthz
- Show /docs page
- Verify data appears

---

## MODULE 7: Building the Frontend Dashboard (90 minutes)

### Video 7.1: Introduction to React (10 min)
**Script Outline:**
- [ ] What is React?
- [ ] Why React for frontend?
- [ ] Component-based architecture
- [ ] Show project structure

**Key Points:**
- "React builds interactive UIs"
- "Components are reusable pieces"

### Video 7.2: Setting Up React Project (15 min)
**Script Outline:**
- [ ] Create React app (or use existing)
- [ ] Show folder structure
- [ ] Explain key files
- [ ] Run development server

**Hands-on:**
- Show project structure
- Explain src/App.js
- Run: `npm start`
- Show default React page

### Video 7.3: Building the Orders Table Component (20 min)
**Script Outline:**
- [ ] Create table component
- [ ] Fetch data from backend API
- [ ] Display orders in table
- [ ] Add styling

**Code Walkthrough:**
```javascript
const [orders, setOrders] = useState([]);

useEffect(() => {
  fetch('/api/analytics')
    .then(res => res.json())
    .then(data => setOrders(data.orders));
}, []);
```

**Key Points:**
- "useState stores component state"
- "useEffect runs on component mount"

### Video 7.4: Adding Auto-Refresh (15 min)
**Script Outline:**
- [ ] Add setInterval for auto-refresh
- [ ] Update every 2 seconds
- [ ] Show loading states
- [ ] Handle errors

**Code Walkthrough:**
```javascript
useEffect(() => {
  const interval = setInterval(() => {
    fetchOrders();
  }, 2000);
  return () => clearInterval(interval);
}, []);
```

**Key Points:**
- "Auto-refresh shows live data"
- "Clean up intervals on unmount"

### Video 7.5: Styling the Dashboard (20 min)
**Script Outline:**
- [ ] Add CSS styling
- [ ] Make it look professional
- [ ] Add colors and layout
- [ ] Responsive design basics

**Hands-on:**
- Add CSS
- Style table
- Add header
- Make it look good

### Video 7.6: Testing Frontend with Backend (10 min)
**Script Outline:**
- [ ] Run both frontend and backend
- [ ] Test complete flow
- [ ] Show live dashboard
- [ ] Verify data updates

**Hands-on:**
- Run backend:8000
- Run frontend:3000
- Show dashboard
- Verify orders appear
- Show auto-refresh working

---

## MODULE 8: Containerization with Docker (75 minutes)

### Video 8.1: Introduction to Docker (10 min)
**Script Outline:**
- [ ] What is Docker?
- [ ] Why containerize?
- [ ] Docker basics: images, containers
- [ ] Docker vs virtual machines

**Key Points:**
- "Docker packages app with dependencies"
- "Runs same everywhere"

### Video 8.2: Creating Dockerfile for Producer (15 min)
**Script Outline:**
- [ ] Create Dockerfile
- [ ] Explain each line
- [ ] Build image
- [ ] Run container

**Code Walkthrough:**
```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY producer.py .
CMD ["python", "producer.py"]
```

**Hands-on:**
- Create Dockerfile
- Build: `docker build -t producer .`
- Run: `docker run producer`

### Video 8.3: Creating Dockerfiles for All Services (20 min)
**Script Outline:**
- [ ] Create Dockerfile for each service
- [ ] Explain differences
- [ ] Build all images
- [ ] Test each container

**Hands-on:**
- Create Dockerfiles for:
  - Producer
  - Fraud service
  - Payment service
  - Analytics service
  - Backend
  - Frontend
- Build each
- Test each

### Video 8.4: Docker Compose for Local Development (20 min)
**Script Outline:**
- [ ] What is Docker Compose?
- [ ] Create docker-compose.yml
- [ ] Define all services
- [ ] Set environment variables
- [ ] Run entire stack locally

**Code Walkthrough:**
```yaml
version: '3.8'
services:
  producer:
    build: ./producer
    environment:
      - KAFKA_BOOTSTRAP_SERVERS=...
  # ... other services
```

**Hands-on:**
- Create docker-compose.yml
- Run: `docker-compose up -d`
- Show all services running
- Test complete system

### Video 8.5: Testing Complete System Locally (10 min)
**Script Outline:**
- [ ] Run docker-compose
- [ ] Verify all services
- [ ] Test end-to-end flow
- [ ] Debug issues

**Hands-on:**
- Start everything
- Check logs
- Test producer → consumers → database → backend → frontend
- Fix any issues

---

## MODULE 9: Introduction to AWS & Infrastructure (60 minutes)

### Video 9.1: Why AWS? (10 min)
**Script Outline:**
- [ ] What is cloud computing?
- [ ] Why AWS?
- [ ] AWS services we'll use
- [ ] Cost considerations

**Key Points:**
- "AWS provides managed services"
- "Pay only for what you use"

### Video 9.2: AWS Services Overview (15 min)
**Script Outline:**
- [ ] ECS (Elastic Container Service)
- [ ] VPC (Virtual Private Cloud)
- [ ] ALB (Application Load Balancer)
- [ ] Secrets Manager
- [ ] CloudWatch
- [ ] Show AWS console

**Visuals:**
- Show AWS console
- Explain each service
- Show how they connect

### Video 9.3: Setting Up AWS Account (15 min)
**Script Outline:**
- [ ] Create AWS account
- [ ] Set up billing alerts
- [ ] Create IAM user
- [ ] Generate access keys
- [ ] Install AWS CLI
- [ ] Configure credentials

**Hands-on:**
- Walk through AWS signup
- Create IAM user
- Generate keys
- Configure: `aws configure`

**Key Points:**
- "Never use root account for daily work"
- "IAM users have limited permissions"

### Video 9.4: Understanding Infrastructure as Code (20 min)
**Script Outline:**
- [ ] What is Infrastructure as Code?
- [ ] Why Terraform?
- [ ] Terraform basics
- [ ] Show Terraform files structure
- [ ] Terraform workflow: init, plan, apply

**Key Points:**
- "Terraform defines infrastructure in code"
- "Version controlled, repeatable"

---

## MODULE 10: Terraform Basics (90 minutes)

### Video 10.1: Terraform Installation & Setup (10 min)
**Script Outline:**
- [ ] Install Terraform
- [ ] Verify installation
- [ ] Create project structure
- [ ] Initialize Terraform

**Hands-on:**
- Install Terraform
- Test: `terraform version`
- Create terraform-ecs folder
- Run: `terraform init`

### Video 10.2: Understanding Terraform Syntax (20 min)
**Script Outline:**
- [ ] Resources
- [ ] Variables
- [ ] Outputs
- [ ] Providers
- [ ] Show example resource

**Code Walkthrough:**
```hcl
resource "aws_ecs_cluster" "this" {
  name = "my-cluster"
}
```

**Key Points:**
- "Resources are AWS components"
- "Variables make it reusable"

### Video 10.3: Creating VPC with Terraform (20 min)
**Script Outline:**
- [ ] What is VPC?
- [ ] Create VPC resource
- [ ] Create subnets
- [ ] Create internet gateway
- [ ] Create NAT gateway

**Code Walkthrough:**
```hcl
module "vpc" {
  source = "terraform-aws-modules/vpc/aws"
  name = "my-vpc"
  cidr = "10.30.0.0/16"
  # ... explain each parameter
}
```

**Hands-on:**
- Create vpc.tf
- Run terraform plan
- Show what will be created

### Video 10.4: Creating Security Groups (15 min)
**Script Outline:**
- [ ] What are security groups?
- [ ] Create ALB security group
- [ ] Create ECS tasks security group
- [ ] Define ingress/egress rules

**Code Walkthrough:**
```hcl
resource "aws_security_group" "alb" {
  ingress {
    from_port = 80
    to_port = 80
    protocol = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

**Key Points:**
- "Security groups are firewalls"
- "Define what traffic is allowed"

### Video 10.5: Setting Up S3 Backend for State (15 min)
**Script Outline:**
- [ ] Why remote state?
- [ ] Create S3 bucket
- [ ] Create DynamoDB table
- [ ] Configure backend
- [ ] Initialize with backend

**Hands-on:**
- Create S3 bucket
- Create DynamoDB table
- Configure backend.tf
- Run terraform init

### Video 10.6: Testing Terraform Configuration (10 min)
**Script Outline:**
- [ ] Run terraform plan
- [ ] Review changes
- [ ] Explain plan output
- [ ] Don't apply yet!

**Hands-on:**
- Run: `terraform plan`
- Show output
- Explain what will be created
- "We'll apply in next module"

---

## MODULE 11: Deploying to AWS ECS (120 minutes)

### Video 11.1: Understanding ECS (15 min)
**Script Outline:**
- [ ] What is ECS?
- [ ] ECS concepts: Cluster, Service, Task Definition
- [ ] Fargate vs EC2
- [ ] Why Fargate?

**Key Points:**
- "ECS runs containers on AWS"
- "Fargate is serverless"

### Video 11.2: Creating ECS Cluster (15 min)
**Script Outline:**
- [ ] Create cluster resource
- [ ] Explain cluster configuration
- [ ] Plan and apply

**Code Walkthrough:**
```hcl
resource "aws_ecs_cluster" "this" {
  name = "${var.project_name}-cluster"
}
```

**Hands-on:**
- Add to ecs.tf
- Run terraform plan
- Apply (if ready)

### Video 11.3: Creating IAM Roles (20 min)
**Script Outline:**
- [ ] What are IAM roles?
- [ ] Create execution role
- [ ] Attach policies
- [ ] Grant Secrets Manager access

**Code Walkthrough:**
```hcl
resource "aws_iam_role" "ecs_task_execution" {
  assume_role_policy = jsonencode({
    Statement = [{
      Principal = { Service = "ecs-tasks.amazonaws.com" }
    }]
  })
}
```

**Key Points:**
- "Roles give permissions to services"
- "Least privilege principle"

### Video 11.4: Setting Up Secrets Manager (20 min)
**Script Outline:**
- [ ] What is Secrets Manager?
- [ ] Create secrets for:
  - Confluent credentials
  - Couchbase credentials
  - GHCR credentials
- [ ] Store secret values

**Code Walkthrough:**
```hcl
resource "aws_secretsmanager_secret" "confluent" {
  name = "${var.project_name}-confluent-credentials"
}
```

**Hands-on:**
- Create secrets.tf
- Add secrets
- Store values (carefully!)

### Video 11.5: Creating Task Definitions (25 min)
**Script Outline:**
- [ ] What is a task definition?
- [ ] Create task for producer
- [ ] Create task for each consumer
- [ ] Create task for backend
- [ ] Create task for frontend
- [ ] Reference secrets

**Code Walkthrough:**
```hcl
resource "aws_ecs_task_definition" "producer" {
  family = "producer"
  cpu = "256"
  memory = "512"
  container_definitions = jsonencode([{
    name = "producer"
    image = var.container_image_producer
    secrets = [{
      name = "KAFKA_BOOTSTRAP_SERVERS"
      valueFrom = "${aws_secretsmanager_secret.confluent.arn}:bootstrap_servers::"
    }]
  }])
}
```

**Key Points:**
- "Task definition = container blueprint"
- "Secrets injected as environment variables"

### Video 11.6: Creating ECS Services (20 min)
**Script Outline:**
- [ ] What is an ECS service?
- [ ] Create service for each task
- [ ] Set desired count
- [ ] Configure networking
- [ ] Attach to load balancer (for web services)

**Code Walkthrough:**
```hcl
resource "aws_ecs_service" "producer" {
  name = "producer-svc"
  cluster = aws_ecs_cluster.this.id
  task_definition = aws_ecs_task_definition.producer.arn
  desired_count = 1
  launch_type = "FARGATE"
}
```

**Hands-on:**
- Create services
- Plan changes
- Explain service vs task

### Video 11.7: Creating Application Load Balancer (15 min)
**Script Outline:**
- [ ] What is ALB?
- [ ] Create ALB
- [ ] Create target groups
- [ ] Create listener
- [ ] Route /api to backend
- [ ] Route / to frontend

**Code Walkthrough:**
```hcl
resource "aws_lb" "ecs_alb" {
  name = "my-alb"
  load_balancer_type = "application"
  subnets = local.public_subnets
}
```

**Key Points:**
- "ALB distributes traffic"
- "Target groups route to services"

---

## MODULE 12: Deploying the Application (90 minutes)

### Video 12.1: Preparing Container Images (20 min)
**Script Outline:**
- [ ] Build Docker images
- [ ] Tag images
- [ ] Push to GitHub Container Registry (GHCR)
- [ ] Verify images

**Hands-on:**
- Build: `docker build -t ghcr.io/username/producer:latest ./producer`
- Push: `docker push ghcr.io/username/producer:latest`
- Show images in GHCR

**Key Points:**
- "Images must be in registry"
- "GHCR is free for public repos"

### Video 12.2: Configuring Terraform Variables (15 min)
**Script Outline:**
- [ ] Review variables.tf
- [ ] Set values.auto.tfvars
- [ ] Create secrets.tfvars
- [ ] Add all required values

**Hands-on:**
- Edit values.auto.tfvars
- Add container image URLs
- Create secrets.tfvars (carefully!)
- Never commit secrets!

### Video 12.3: Running Terraform Plan (15 min)
**Script Outline:**
- [ ] Run terraform plan
- [ ] Review all changes
- [ ] Explain what will be created
- [ ] Check for errors
- [ ] Fix any issues

**Hands-on:**
- Run: `terraform plan -var-file="secrets.tfvars"`
- Review output
- Explain resources
- Fix errors

### Video 12.4: Deploying with Terraform Apply (20 min)
**Script Outline:**
- [ ] Run terraform apply
- [ ] Watch resources being created
- [ ] Wait for completion
- [ ] Get outputs (ALB URL)

**Hands-on:**
- Run: `terraform apply -var-file="secrets.tfvars"`
- Watch progress
- Show resources being created
- Get ALB DNS name

**Key Points:**
- "First deployment takes 10-15 minutes"
- "Be patient!"

### Video 12.5: Verifying Deployment (20 min)
**Script Outline:**
- [ ] Check ECS services status
- [ ] View CloudWatch logs
- [ ] Test ALB endpoint
- [ ] Verify all services running
- [ ] Test complete flow

**Hands-on:**
- Check ECS console
- View logs
- Test: `curl http://alb-dns-name`
- Verify frontend loads
- Verify backend API works
- Check dashboard shows data

---

## MODULE 13: Monitoring & Troubleshooting (60 minutes)

### Video 13.1: CloudWatch Logs (15 min)
**Script Outline:**
- [ ] What is CloudWatch?
- [ ] View logs for each service
- [ ] Search logs
- [ ] Filter by service

**Hands-on:**
- Open CloudWatch
- Show log groups
- View producer logs
- View consumer logs
- Search for errors

### Video 13.2: ECS Service Health (15 min)
**Script Outline:**
- [ ] Check service status
- [ ] View running tasks
- [ ] Check task health
- [ ] View task logs

**Hands-on:**
- Open ECS console
- Check each service
- View tasks
- Check health status

### Video 13.3: Common Issues & Solutions (20 min)
**Script Outline:**
- [ ] Service won't start
- [ ] Connection errors
- [ ] Secrets not working
- [ ] Health check failures
- [ ] Debugging tips

**Common Issues:**
- "Task failed to start" → Check logs
- "Connection refused" → Check security groups
- "Secrets not found" → Check IAM permissions
- "Health check failing" → Check endpoint

### Video 13.4: Setting Up Alarms (10 min)
**Script Outline:**
- [ ] Create CloudWatch alarms
- [ ] Monitor ALB errors
- [ ] Monitor latency
- [ ] Set up notifications (optional)

**Hands-on:**
- Create alarm for 5xx errors
- Create alarm for high latency
- Show alarm configuration

---

## MODULE 14: Advanced Topics (Optional - 60 minutes)

### Video 14.1: Scaling Services (15 min)
**Script Outline:**
- [ ] Manual scaling
- [ ] Auto-scaling basics
- [ ] Update desired count
- [ ] Cost considerations

**Hands-on:**
- Update desired_count
- Apply changes
- Show scaling in action

### Video 14.2: Updating the Application (15 min)
**Script Outline:**
- [ ] Update code
- [ ] Build new image
- [ ] Push to registry
- [ ] Force new deployment
- [ ] Zero-downtime updates

**Hands-on:**
- Make code change
- Build new image
- Push image
- Force ECS service update
- Show rolling update

### Video 14.3: Cost Optimization (15 min)
**Script Outline:**
- [ ] Review AWS costs
- [ ] Cost breakdown
- [ ] Optimization tips
- [ ] Free tier limits
- [ ] Cleanup resources

**Key Points:**
- "Fargate pricing"
- "ALB costs"
- "NAT Gateway costs"
- "How to reduce costs"

### Video 14.4: Cleanup & Destroy (15 min)
**Script Outline:**
- [ ] When to destroy
- [ ] Run terraform destroy
- [ ] Verify cleanup
- [ ] Cost savings

**Hands-on:**
- Run: `terraform destroy`
- Confirm deletion
- Verify resources deleted
- Check costs

**Warning:**
- "This deletes everything!"
- "Only do this when done"

---

## MODULE 15: Course Wrap-up (30 minutes)

### Video 15.1: What We Built (10 min)
**Script Outline:**
- [ ] Recap entire system
- [ ] Show architecture
- [ ] Highlight key learnings
- [ ] Real-world applications

**Visuals:**
- Show complete architecture
- Highlight each component
- Show live system

### Video 15.2: Next Steps (10 min)
**Script Outline:**
- [ ] What to learn next
- [ ] Additional features to add
- [ ] Production considerations
- [ ] Resources for further learning

**Suggestions:**
- Add authentication
- Add more consumers
- Implement monitoring
- Add CI/CD pipeline
- Learn Kubernetes

### Video 15.3: Final Q&A & Thank You (10 min)
**Script Outline:**
- [ ] Answer common questions
- [ ] Thank students
- [ ] Encourage to build more
- [ ] Share your projects

---

## Course Materials Structure

```
udemy-course/
├── 01-introduction/
│   ├── 1.1-welcome.mp4
│   ├── 1.2-architecture.mp4
│   └── 1.3-setup.mp4
├── 02-kafka-basics/
│   ├── 2.1-what-is-kafka.mp4
│   ├── 2.2-concepts.mp4
│   └── 2.3-confluent-setup.mp4
├── 03-producer/
│   ├── 3.1-overview.mp4
│   ├── 3.2-code-part1.mp4
│   ├── 3.3-code-part2.mp4
│   └── 3.4-testing.mp4
├── ... (continue for all modules)
├── resources/
│   ├── project-code/
│   ├── diagrams/
│   ├── cheatsheets/
│   └── scripts/
└── README.md
```

---

## Video Recording Tips

1. **Screen Recording:**
   - Use OBS Studio or Camtasia
   - Record at 1080p minimum
   - Show code clearly (zoom if needed)
   - Use cursor highlights

2. **Audio:**
   - Use good microphone
   - Record in quiet space
   - Remove background noise
   - Speak clearly and slowly

3. **Editing:**
   - Remove long pauses
   - Add text overlays for key points
   - Add arrows/highlights for important parts
   - Keep videos under 20 minutes

4. **Structure:**
   - Start with what you'll learn
   - Show the code/UI
   - Explain each step
   - End with summary

5. **Human Touch:**
   - Make mistakes (then fix them!)
   - Show real debugging
   - Share personal tips
   - Be conversational

---

## Learning Objectives Summary

By the end of this course, students will be able to:

✅ Understand Kafka and real-time streaming  
✅ Build Kafka producers and consumers in Python  
✅ Work with Couchbase NoSQL database  
✅ Create REST APIs with FastAPI  
✅ Build React frontend applications  
✅ Containerize applications with Docker  
✅ Deploy to AWS using Terraform  
✅ Monitor and troubleshoot cloud applications  
✅ Build complete end-to-end systems  

---

## Prerequisites

- Basic Python programming
- Familiarity with command line
- Basic understanding of web development (helpful but not required)
- No prior Kafka, AWS, or Terraform experience needed!

---

## Course Completion Certificate

Students who complete all modules and deploy their application will receive a certificate of completion.

---

**Good luck creating your Udemy course! This structure provides a comprehensive, beginner-friendly path from zero to deployed application.**


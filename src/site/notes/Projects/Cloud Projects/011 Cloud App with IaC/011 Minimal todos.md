---
{"dg-publish":true,"permalink":"/projects/cloud-projects/011-cloud-app-with-ia-c/011-minimal-todos/"}
---

#projects 

On this project i want to explore the container offering using aws, use the basics of terraform to be able to provision the infrastructure declaratively

**Goals** 
Implement a minimal version of a todos application, create containers, apply terraform 
basics to provision the required infrastructure and a simple pipeline to automate releases

**Key points:** Terraform basics, cloud deployments, containerization, minimal CI/CD.

 **Infrastructure breakdown**
 - 1 VPC
 - 4 subnets, 2 public subnets and 2 private subnets
 - 1 ALB
 - 1 Internet gateway
 - 1 NAT gateway
 - 1 ECS Task
 - 1 ECS Cluster

**Architecture diagram**
![aws.drawio.jpg](/img/user/Projects/Cloud%20Projects/011%20Cloud%20App%20with%20IaC/aws.drawio.jpg)

**Process**

I got to try V0 in a recent work trip and I asked it to create a simple next.js app, i did a really good job i think I created a different sructured compared to some other project starters I've seen before but nonetheless It did a good job for the instructions I gave it

After I connected it to a Neon postgresql db (I'm saving having to setup the db inside aws for another project) I was able to test it locally 

Now that it worked locally I created a docker compose and dockerfile to create the container that  would be deployed using ECS

docker-compose.yml
```yaml
version: "3.9"
services:
frontend:
build: ./frontend
container_name: cloud-app-frontend
platform: linux/amd64 # <-- to prepare to run on AWS

ports:
	- "3000:3000"

environment:
	- DATABASE_URL=

volumes:
	- ./frontend:/app
	- /app/node_modules # <-- prevents overwriting node_modules
```

Dockerfile
```Dockerfile
FROM node:22.11-slim 

# Set work directory
WORKDIR /app

# Install dependencies
COPY package.json pnpm-lock.yaml ./
RUN npm install -g pnpm@latest-10
RUN pnpm install --frozen-lockfile

# Copy app
COPY . .

# Expose port
EXPOSE 3000

# Run server
CMD ["pnpm", "run", "dev"]
```

> Note: the app is running still in development mode, in a proper production environment the app should be build and then run using the start command

Once I got the next.js app running using a container the next step was defining the infrastructure with terraform, I remember the basic and also checked the [[Resources/Books/Terraform up & running notes\|Terraform up & running notes]]

First creating the VPC, subnets and route table

```hcl
variable "aws_region" {
  description = "The AWS region to deploy resources in"
  type = string
  default = "us-east-1"
}
provider "aws" {
  region = var.aws_region
  profile = "terraform-dev"
}

# Networking (VPC, Subnets, IGW, Routes)

resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_support   = true
  enable_dns_hostnames = true
}

resource "aws_internet_gateway" "igw" {
  vpc_id = aws_vpc.main.id
}

resource "aws_subnet" "public_a" {
  vpc_id = aws_vpc.main.id
  cidr_block = "10.0.1.0/24"
  availability_zone = "${var.aws_region}a"
  map_public_ip_on_launch = true
}

resource "aws_subnet" "public_b" {
  vpc_id = aws_vpc.main.id
  cidr_block = "10.0.2.0/24"
  availability_zone = "${var.aws_region}b"
  map_public_ip_on_launch = true
}

resource "aws_subnet" "private_a" {
  vpc_id = aws_vpc.main.id
  cidr_block = "10.0.3.0/24"
  availability_zone = "${var.aws_region}a"
}

resource "aws_subnet" "private_b" {
  vpc_id = aws_vpc.main.id
  cidr_block = "10.0.4.0/24"
  availability_zone = "${var.aws_region}b"
}
```

when creating multiple subnets on one az I think it's easier to use *cidrsubnet* but with just one maybe it's clearer to do it directly with the CIDR block string 

**Route Table vs. Security Group**
They serve **different purposes**:
- **Route Table** = _where packets go_. It controls whether instances in a subnet can reach the internet (via IGW) or not. Without it, your subnet is isolated.
    - In our setup: attaching the route table with a `0.0.0.0/0 → IGW` is enough to make the subnet _public_.
- **Security Group** = _who is allowed to talk to you_. It acts as a firewall on the ECS tasks/ALB.
    - In our setup: the SG allows HTTP (port 80) inbound.

So yes — the public subnets **need both**:
- Route Table → for Internet connectivity.
- SG → for access control.

Now we need to route the traffic
```hcl
resource "aws_route_table" "public_internet" {
  vpc_id = aws_vpc.main.id
}

resource "aws_route" "public_internet_access" {
  route_table_id         = aws_route_table.public_internet.id
  destination_cidr_block = "0.0.0.0/0"
  gateway_id             = aws_internet_gateway.igw.id
}

resource "aws_route_table_association" "public_a" {
  subnet_id = aws_subnet.public_a.id
  route_table_id = aws_route_table.public_internet.id
}

resource "aws_route_table_association" "public_b" {
  subnet_id = aws_subnet.public_b.id
  route_table_id = aws_route_table.public_internet.id
}
```

The *public_internet_access* route allows the public subnet to access the internet and because the NAT gateway will be deploy in those subnets, will allow the private subnets to access the internet as well

The ecs tasks need an IAM role with permissions 

```hcl
# IAM Role for ECS Tasks
# is creating a new rola that has the policy to execute ecs tasks?

resource "aws_iam_role" "ecs_task_execution_role" {
  name = "ecsTaskExecutionRole"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = "sts:AssumeRole"
      Effect = "Allow"
      Principal = {
        Service = "ecs-tasks.amazonaws.com"
      }
    }]
  })
}

resource "aws_iam_role_policy_attachment" "ecs_task_execution_role_policy" {
  role = aws_iam_role.ecs_task_execution_role.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy"
}
```

Then we create the ECS cluster and tasks 
```
# ECS Cluster
resource "aws_ecs_cluster" "main" {
  name = "demo-cluster"
}

# ECS task
resource "aws_ecs_task_definition" "nginx" {
  family = "nginx-task" #?
  network_mode = "awsvpc"
  requires_compatibilities = ["FARGATE"]
  cpu = "512"
  memory = "1024"
  execution_role_arn = aws_iam_role.ecs_task_execution_role.arn
  container_definitions = jsonencode([{
    name = "nginx"
    iamge = "XXXXXXXXX.dkr.ecr.us-east-1.amazonaws.com/cloud/todo:latest"
    essential = true
    portMappings =  [{
      containerPort = 3000
      hostPort = 3000
      protocol = "tcp"
    }]
  }])
}
```


Now, to direct the traffic on the vpc we need 2 security groups, one for the tasks and another one for the ALB
```hcl
resource "aws_security_group" "alb" {
  vpc_id = aws_vpc.main.id
  
  ingress {
    from_port = 3000
    to_port = 3000
    protocol = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  egress {
    from_port = 0
    to_port = 0
    protocol = "-1" # -1 ?
    cidr_blocks = ["0.0.0.0/0"]
  }
}

resource "aws_security_group" "ecs_tasks" {
  vpc_id = aws_vpc.main.id
  ingress {
    from_port = 3000
    to_port = 3000
    protocol = "tcp"
    security_groups = [aws_security_group.alb.id]
  }
  egress {
    from_port = 0
    to_port = 0
    protocol = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

the *ecs_tasks* security group allows connections only through the ALB

Now we can create the ALB
```hcl
# ALB
resource "aws_alb" "ecs" {
  name = "ecs-alb"
  load_balancer_type = "application"
  subnets = [aws_subnet.public_a.id, aws_subnet.public_b.id]
  security_groups = [aws_security_group.alb.id]
}

resource "aws_alb_target_group" "ecs" { # ?? what does a target group do?
  name = "ecs-tg"
  port = 3000
  protocol = "HTTP"
  vpc_id = aws_vpc.main.id
  health_check {
    path = "/api/health"
    interval = 30
    timeout = 5
    healthy_threshold = 2
    unhealthy_threshold = 2
    matcher = "200"
  }
}

resource "aws_alb_listener" "http" { # ?? what does a listener do?
  load_balancer_arn = aws_alb.ecs.arn
  port = 3000
  protocol = "HTTP"
  default_action {
    type = "forward"
    target_group_arn = aws_alb_target_group.ecs.arn
  }
}
```

finally the final step is to create the ECS service

```hcl
resource "aws_ecs_service" "nginx" {
  name = "nginx-service"
  cluster = aws_ecs_cluster.main.id
  task_definition = aws_ecs_task_definition.nginx.arn
  desired_count = 2
  launch_type = "FARGATE"

  network_configuration {
    subnets = [aws_subnet.private_a.id, aws_subnet.private_b.id]
    assign_public_ip = false
    security_groups = [aws_security_group.ecs_tasks.id] # why would you use multiple security groups
  }

  load_balancer {
    target_group_arn = aws_alb_target_group.ecs.arn
    container_name = "nginx"
    container_port = 80
  }

  depends_on = [aws_alb.ecs]
}
```


Executing terraform plan i see the 26 components terraform would provision 
the ALB was the resource that took the most when creating 

![Screenshot 2025-08-30 at 5.24.07 PM.png](/img/user/Projects/Cloud%20Projects/011%20Cloud%20App%20with%20IaC/Screenshot%202025-08-30%20at%205.24.07%20PM.png)

I find useful the graphic representation of the vpc's structure 
![Screenshot 2025-08-31 at 6.06.57 PM.png](/img/user/Projects/Cloud%20Projects/011%20Cloud%20App%20with%20IaC/Screenshot%202025-08-31%20at%206.06.57%20PM.png)

also, once the ALB is created, there is also a similar helper graph to see the specific targets the load balancer has at that moment 
![Screenshot 2025-08-31 at 5.58.00 PM.png](/img/user/Screenshot%202025-08-31%20at%205.58.00%20PM.png)

After visiting the ALB dns name on port 300 i was able to see the app running 
![Screenshot 2025-08-31 at 5.57.28 PM.png](/img/user/Screenshot%202025-08-31%20at%205.57.28%20PM.png)

I'm working on an AWS certification and it was really useful to practice this exercise, it helped to solidify my understanding of ecs and networking

Of course there were a lot of missing details to make this a production ready exercise
- use correct commands inside the Dockerfile
- implement a pipeline to create new ECR images when the app changes and it's ready for release
- use a proper DB option inside AWS and include it on the infrastructure plan
- not for production but to add names to all elements and tags for billing 

**Issues i found, to remember**
I got an error creating the IAM Role, one already existed with the same name, now my question, i think some of the services were created if i change the terraform file, terraform should be able to keep creating the infrastructure 

![Screenshot 2025-08-30 at 5.30.04 PM.png](/img/user/Projects/Cloud%20Projects/011%20Cloud%20App%20with%20IaC/Screenshot%202025-08-30%20at%205.30.04%20PM.png)

![Screenshot 2025-08-30 at 5.32.53 PM.png](/img/user/Projects/Cloud%20Projects/011%20Cloud%20App%20with%20IaC/Screenshot%202025-08-30%20at%205.32.53%20PM.png)

Terrraform was able to keep track that just 3 components were not created the first time and then just create the ones that were missing 

One of the error messages i got was 
> The provided target group * has target type instance, which is incompatible with the awsvpc network mode specified in the task definition.

*Investigate in detail*
When you run **Fargate tasks with `awsvpc` network mode**, the ECS service **registers ENIs (elastic network interfaces)** directly with the ALB target group.

👉 That means your **target group must use `target_type = "ip"`**, not the default `"instance"`.

Another issue I has was that i forgot to associate the public subnet with the NAT gateways with the internet gateway so the private subnets were not able to connect to the internet 


I enjoy making this project, hopefully is a first version to build upon and practice new things 



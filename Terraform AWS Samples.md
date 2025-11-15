# Terraform AWS Samples
### Start with provider & basic variables (put this at top of your file):
```
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
  required_version = ">= 1.0.0"
}

provider "aws" {
  region = var.aws_region
}

variable "aws_region" {
  type    = string
  default = "us-east-1"
}

variable "project" {
  type    = string
  default = "devops-sample"
}
```
### 1) VPC + Subnet + Internet Gateway + Route Table + NAT Gateway
```
# VPC
resource "aws_vpc" "this" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true
  tags = { Name = "${var.project}-vpc" }
}

# Public Subnet (for IGW, NAT, ALB, etc)
resource "aws_subnet" "public" {
  vpc_id                  = aws_vpc.this.id
  cidr_block              = "10.0.1.0/24"
  map_public_ip_on_launch = true
  availability_zone       = "${var.aws_region}a"
  tags = { Name = "${var.project}-public-subnet" }
}

# Private Subnet (for EC2/RDS/ECS tasks)
resource "aws_subnet" "private" {
  vpc_id            = aws_vpc.this.id
  cidr_block        = "10.0.2.0/24"
  availability_zone = "${var.aws_region}a"
  tags = { Name = "${var.project}-private-subnet" }
}

# Internet Gateway
resource "aws_internet_gateway" "igw" {
  vpc_id = aws_vpc.this.id
  tags = { Name = "${var.project}-igw" }
}

# Public Route Table & association
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.this.id
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.igw.id
  }
  tags = { Name = "${var.project}-public-rt" }
}

resource "aws_route_table_association" "public_assoc" {
  subnet_id      = aws_subnet.public.id
  route_table_id = aws_route_table.public.id
}

# Elastic IP for NAT
resource "aws_eip" "nat_eip" {
  vpc = true
  tags = { Name = "${var.project}-nat-eip" }
}

# NAT Gateway (in public subnet)
resource "aws_nat_gateway" "nat" {
  allocation_id = aws_eip.nat_eip.id
  subnet_id     = aws_subnet.public.id
  tags = { Name = "${var.project}-nat" }
}

# Private route to NAT for outbound internet
resource "aws_route_table" "private" {
  vpc_id = aws_vpc.this.id
  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.nat.id
  }
  tags = { Name = "${var.project}-private-rt" }
}

resource "aws_route_table_association" "private_assoc" {
  subnet_id      = aws_subnet.private.id
  route_table_id = aws_route_table.private.id
}
```
### 2) Security Group (example for web + ALB)
```
# Web Security Group (allow HTTP/HTTPS from ALB)
resource "aws_security_group" "web_sg" {
  name        = "${var.project}-web-sg"
  description = "Allow HTTP from ALB and outbound to internet"
  vpc_id      = aws_vpc.this.id

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]  # if directly open; prefer ALB SG referencing below
  }

  ingress {
    from_port       = 443
    to_port         = 443
    protocol        = "tcp"
    cidr_blocks     = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = { Name = "${var.project}-web-sg" }
}

# ALB Security Group (allows internet -> ALB; ALB -> web on 80)
resource "aws_security_group" "alb_sg" {
  name   = "${var.project}-alb-sg"
  vpc_id = aws_vpc.this.id

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port       = 80
    to_port         = 80
    protocol        = "tcp"
    security_groups = [aws_security_group.web_sg.id]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = { Name = "${var.project}-alb-sg" }
}
```
### 3) EC2 (example with user-data)
```
resource "aws_instance" "web" {
  ami                         = "ami-0c94855ba95c71c99" # replace with correct region AMI
  instance_type               = "t3.micro"
  subnet_id                   = aws_subnet.private.id
  vpc_security_group_ids      = [aws_security_group.web_sg.id]
  associate_public_ip_address = false

  user_data = <<-EOF
              #!/bin/bash
              yum update -y
              yum install -y httpd
              systemctl enable httpd
              systemctl start httpd
              echo "Hello from ${var.project}" > /var/www/html/index.html
              EOF

  tags = { Name = "${var.project}-ec2-web" }
}
```
### 4) S3 Bucket (with versioning and public-block)
```
resource "aws_s3_bucket" "app_bucket" {
  bucket = "${var.project}-bucket-${substr(md5(var.project),0,6)}"
  acl    = "private"

  versioning {
    enabled = true
  }

  server_side_encryption_configuration {
    rule {
      apply_server_side_encryption_by_default {
        sse_algorithm = "AES256"
      }
    }
  }

  tags = { Name = "${var.project}-s3" }
}

resource "aws_s3_bucket_public_access_block" "block" {
  bucket = aws_s3_bucket.app_bucket.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}
```
### 5) IAM: User / Role / Policy (examples)
```
# IAM User (programmatic access)
resource "aws_iam_user" "dev_user" {
  name = "${var.project}-user"
  tags = { Name = "${var.project}-user" }
}

resource "aws_iam_access_key" "dev_user_key" {
  user = aws_iam_user.dev_user.name
}

# Inline policy or managed policy example (S3 read-only)
resource "aws_iam_policy" "s3_readonly" {
  name        = "${var.project}-s3-readonly"
  description = "S3 read-only for examples"
  policy      = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action   = ["s3:GetObject","s3:ListBucket"]
      Effect   = "Allow"
      Resource = ["${aws_s3_bucket.app_bucket.arn}","${aws_s3_bucket.app_bucket.arn}/*"]
    }]
  })
}

resource "aws_iam_user_policy_attachment" "attach_s3" {
  user       = aws_iam_user.dev_user.name
  policy_arn = aws_iam_policy.s3_readonly.arn
}

# IAM Role for EC2 (example assume role for ec2)
resource "aws_iam_role" "ec2_role" {
  name = "${var.project}-ec2-role"
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = "sts:AssumeRole"
      Effect = "Allow"
      Principal = { Service = "ec2.amazonaws.com" }
    }]
  })
}

resource "aws_iam_role_policy_attachment" "ec2_s3_attach" {
  role       = aws_iam_role.ec2_role.name
  policy_arn = aws_iam_policy.s3_readonly.arn
}
```
## 6) Load Balancers — ALB and NLB
### ALB (Application Load Balancer)
```
resource "aws_lb" "alb" {
  name               = "${var.project}-alb"
  load_balancer_type = "application"
  subnets            = [aws_subnet.public.id]
  security_groups    = [aws_security_group.alb_sg.id]
  tags = { Name = "${var.project}-alb" }
}

resource "aws_lb_target_group" "tg" {
  name     = "${var.project}-tg"
  port     = 80
  protocol = "HTTP"
  vpc_id   = aws_vpc.this.id
  health_check {
    path                = "/"
    matcher             = "200-399"
    interval            = 30
    timeout             = 5
    healthy_threshold   = 2
    unhealthy_threshold = 2
  }
}

resource "aws_lb_listener" "http" {
  load_balancer_arn = aws_lb.alb.arn
  port              = "80"
  protocol          = "HTTP"

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.tg.arn
  }
}

# Register EC2 instance to target group
resource "aws_lb_target_group_attachment" "web_attach" {
  target_group_arn = aws_lb_target_group.tg.arn
  target_id        = aws_instance.web.id
  port             = 80
}
```
### NLB (Network Load Balancer)
```
resource "aws_lb" "nlb" {
  name               = "${var.project}-nlb"
  load_balancer_type = "network"
  subnets            = [aws_subnet.public.id]
  tags = { Name = "${var.project}-nlb" }
}

resource "aws_lb_target_group" "nlb_tg" {
  name     = "${var.project}-nlb-tg"
  port     = 80
  protocol = "TCP"
  vpc_id   = aws_vpc.this.id
}
```
### 7) RDS (MySQL example, in private subnets)
```
resource "aws_db_subnet_group" "rds_subnet" {
  name       = "${var.project}-rds-subnet"
  subnet_ids = [aws_subnet.private.id]
  tags = { Name = "${var.project}-rds-subnet" }
}

resource "aws_db_instance" "mysql" {
  allocated_storage    = 20
  engine               = "mysql"
  engine_version       = "8.0"
  instance_class       = "db.t3.micro"
  name                 = "mydb"
  username             = "admin"
  password             = "ChangeMe123!"
  skip_final_snapshot  = true
  db_subnet_group_name = aws_db_subnet_group.rds_subnet.name
  vpc_security_group_ids = [aws_security_group.web_sg.id]
  tags = { Name = "${var.project}-rds" }
}
```
### 8) DynamoDB Table
```
resource "aws_dynamodb_table" "table" {
  name         = "${var.project}-table"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "id"
  attribute {
    name = "id"
    type = "S"
  }
  tags = { Name = "${var.project}-dynamo" }
}
```
### 9) Lambda (with role + CloudWatch LogGroup)
```
# Role for Lambda with basic execution
resource "aws_iam_role" "lambda_role" {
  name = "${var.project}-lambda-role"
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = "sts:AssumeRole"
      Effect = "Allow"
      Principal = { Service = "lambda.amazonaws.com" }
    }]
  })
}

resource "aws_iam_role_policy_attachment" "lambda_basic_attach" {
  role       = aws_iam_role.lambda_role.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole"
}

# Lambda function (local zip "function.zip" must exist)
resource "aws_lambda_function" "fn" {
  function_name = "${var.project}-fn"
  filename      = "function.zip"     # or use s3_bucket & s3_key
  handler       = "index.handler"
  runtime       = "nodejs18.x"
  role          = aws_iam_role.lambda_role.arn
  timeout       = 10
  memory_size   = 128
}

# CloudWatch Log Group for Lambda (optional — created automatically on first run but it's okay to define)
resource "aws_cloudwatch_log_group" "lambda_logs" {
  name              = "/aws/lambda/${aws_lambda_function.fn.function_name}"
  retention_in_days = 14
}
```
### 10) API Gateway (REST) + Lambda Integration
```
# API Gateway REST API
resource "aws_api_gateway_rest_api" "api" {
  name = "${var.project}-api"
}

# root resource
data "aws_api_gateway_resource" "root" {
  rest_api_id = aws_api_gateway_rest_api.api.id
  path        = "/"
}

# create resource "hello"
resource "aws_api_gateway_resource" "hello" {
  rest_api_id = aws_api_gateway_rest_api.api.id
  parent_id   = aws_api_gateway_rest_api.api.root_resource_id
  path_part   = "hello"
}

resource "aws_api_gateway_method" "hello_get" {
  rest_api_id   = aws_api_gateway_rest_api.api.id
  resource_id   = aws_api_gateway_resource.hello.id
  http_method   = "GET"
  authorization = "NONE"
}

resource "aws_api_gateway_integration" "hello_integration" {
  rest_api_id = aws_api_gateway_rest_api.api.id
  resource_id = aws_api_gateway_resource.hello.id
  http_method = aws_api_gateway_method.hello_get.http_method
  integration_http_method = "POST"
  type                     = "AWS_PROXY"
  uri                      = aws_lambda_function.fn.invoke_arn
}

# deploy stage
resource "aws_api_gateway_deployment" "deployment" {
  depends_on = [aws_api_gateway_integration.hello_integration]
  rest_api_id = aws_api_gateway_rest_api.api.id
  stage_name  = "dev"
}

# permission for API Gateway to invoke Lambda
resource "aws_lambda_permission" "apigw_lambda" {
  statement_id  = "AllowAPIGatewayInvoke"
  action        = "lambda:InvokeFunction"
  function_name = aws_lambda_function.fn.function_name
  principal     = "apigateway.amazonaws.com"
  source_arn    = "${aws_api_gateway_rest_api.api.execution_arn}/*/*"
}
```
### 11) CloudWatch Logs (create log group and an example metric filter)
```
resource "aws_cloudwatch_log_group" "app_logs" {
  name              = "/${var.project}/app"
  retention_in_days = 14
}

resource "aws_cloudwatch_metric_filter" "error_count" {
  name           = "${var.project}-errors"
  log_group_name = aws_cloudwatch_log_group.app_logs.name
  pattern        = "\"ERROR\""
  metric_transformation {
    name      = "${var.project}_errors"
    namespace = var.project
    value     = "1"
  }
}
```
## 12) ECS Cluster + Task + Service + ECR repository
### ECR repository
```
resource "aws_ecr_repository" "app_repo" {
  name = "${var.project}-repo"
  image_scanning_configuration { scan_on_push = true }
  tags = { Name = "${var.project}-ecr" }
}
```
### ECS cluster
```
resource "aws_ecs_cluster" "cluster" {
  name = "${var.project}-cluster"
}
```
### IAM roles for ECS task/execution
```
resource "aws_iam_role" "ecs_task_execution" {
  name = "${var.project}-ecs-exec"
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = "sts:AssumeRole"
      Effect = "Allow"
      Principal = { Service = "ecs-tasks.amazonaws.com" }
    }]
  })
}

resource "aws_iam_role_policy_attachment" "ecs_task_execution_attach" {
  role       = aws_iam_role.ecs_task_execution.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy"
}
```
### Task definition (Fargate example)
```
resource "aws_ecs_task_definition" "task" {
  family                   = "${var.project}-task"
  network_mode             = "awsvpc"
  requires_compatibilities = ["FARGATE"]
  cpu                      = "256"
  memory                   = "512"
  execution_role_arn       = aws_iam_role.ecs_task_execution.arn
  container_definitions = jsonencode([
    {
      name  = "app"
      image = "${aws_ecr_repository.app_repo.repository_url}:latest" # you must push image
      essential = true
      portMappings = [{ containerPort = 80, hostPort = 80 }]
    }
  ])
}
```
### ECS Service
```
resource "aws_ecs_service" "service" {
  name            = "${var.project}-service"
  cluster         = aws_ecs_cluster.cluster.id
  task_definition = aws_ecs_task_definition.task.arn
  launch_type     = "FARGATE"
  desired_count   = 1
  network_configuration {
    subnets         = [aws_subnet.private.id]
    security_groups = [aws_security_group.web_sg.id]
    assign_public_ip = false
  }
  depends_on = [aws_lb_listener.http]
}
```
### Notes & next steps

* Replace placeholders: AMI IDs, function.zip (for Lambda), credentials/secrets (never commit), and any hard-coded passwords. Prefer variables or remote state for secrets.

* For production, separate networking, compute, and IAM into modules and use terraform workspace or separate state files.

* Use terraform fmt and terraform validate before terraform plan.

* For RDS and other stateful services, add final_snapshot settings and proper backups.

* To push images to ECR, use aws ecr get-login-password or CI pipeline steps.


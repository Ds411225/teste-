# Teste -Codigos de shapes do projeto e Rede 
Banco Carrefour 


provider "aws" {
  region = "us-east-1"
}

# Criar a VPC
resource "aws_vpc" "xpto_vpc" {
  cidr_block = "10.0.0.0/16"
  enable_dns_support = true
  enable_dns_hostnames = true
  tags = { Name = "xpto_vpc" }
}

# Criar Subnets
resource "aws_subnet" "public_subnet" {
  vpc_id                  = aws_vpc.xpto_vpc.id
  cidr_block              = "10.0.1.0/24"
  map_public_ip_on_launch = true
  availability_zone       = "us-east-1a"
  tags = { Name = "public_subnet" }
}

resource "aws_subnet" "private_subnet" {
  vpc_id                  = aws_vpc.xpto_vpc.id
  cidr_block              = "10.0.2.0/24"
  map_public_ip_on_launch = false
  availability_zone       = "us-east-1b"
  tags = { Name = "private_subnet" }
}

# Criar Gateway de Internet
resource "aws_internet_gateway" "xpto_igw" {
  vpc_id = aws_vpc.xpto_vpc.id
  tags = { Name = "xpto_igw" }
}

# Criar tabela de rotas para internet
resource "aws_route_table" "public_rt" {
  vpc_id = aws_vpc.xpto_vpc.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.xpto_igw.id
  }

  tags = { Name = "public_rt" }
}

resource "aws_route_table_association" "public_association" {
  subnet_id      = aws_subnet.public_subnet.id
  route_table_id = aws_route_table.public_rt.id
}

# Criar Security Group para EC2
resource "aws_security_group" "xpto_sg" {
  vpc_id = aws_vpc.xpto_vpc.id

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = { Name = "xpto_sg" }
}

# Criar EC2 para serviços
resource "aws_instance" "xpto_ec2" {
  ami           = "ami-0c55b159cbfafe1f0"  # Substituir por uma AMI válida
  instance_type = "m5.large"
  subnet_id     = aws_subnet.public_subnet.id
  security_groups = [aws_security_group.xpto_sg.name]

  tags = { Name = "xpto_ec2" }
}

# Criar Cluster EKS
resource "aws_eks_cluster" "xpto_eks" {
  name     = "xpto-cluster"
  role_arn = aws_iam_role.eks_role.arn
  vpc_config {
    subnet_ids = [aws_subnet.public_subnet.id, aws_subnet.private_subnet.id]
  }
}

# Criar banco de dados RDS
resource "aws_db_instance" "xpto_rds" {
  allocated_storage    = 20
  storage_type         = "gp2"
  engine              = "postgres"
  instance_class      = "db.t3.medium"
  identifier         = "xpto-rds"
  username           = "admin"
  password           = "changeme123"  # Use variáveis seguras no Terraform!
  multi_az           = true
  publicly_accessible = false
  vpc_security_group_ids = [aws_security_group.xpto_sg.id]
  db_subnet_group_name = aws_db_subnet_group.xpto_subnet_group.name
}

# Criar S3 para armazenamento
resource "aws_s3_bucket" "xpto_s3" {
  bucket = "xpto-bucket"
  acl    = "private"
  tags = { Name = "xpto_s3" }
}

# Criar AWS Backup Plan
resource "aws_backup_plan" "xpto_backup" {
  name = "xpto_backup_plan"
  rule {
    rule_name         = "daily_backup"
    target_vault_name = "Default"
    schedule          = "cron(0 12 * * ? *)"
  }
}

# Criar IAM Role para EKS
resource "aws_iam_role" "eks_role" {
  name = "eks-role"
  assume_role_policy = <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "eks.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF
}

# Criar IAM Role para EC2
resource "aws_iam_role" "ec2_role" {
  name = "ec2-role"
  assume_role_policy = <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "ec2.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF
}

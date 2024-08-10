# Deploy to Kubernetes

This repository contains a project for deploying a frontend application to Kubernetes using GitHub Actions for continuous integration and deployment. The application is built using Docker and deployed to Amazon Elastic Kubernetes Service (EKS).

## Table of Contents

- [Features](#features)
- [Technologies Used](#technologies-used)
- [Getting Started](#getting-started)
- [Workflow Overview](#workflow-overview)
- [Deployment](#deployment)
- [Terraform Configuration](#Terraform-Configuration)
- [Usage](#usage)
- [Contributing](#contributing)
- [License](#license)
## Features

- Automated deployment of the frontend application to a Kubernetes cluster.
- Integration with Docker for containerization.
- Continuous deployment using GitHub Actions.
- Supports AWS EKS for scalable container orchestration.

## Technologies Used

- **Containerization:** Docker
- **Orchestration:** Kubernetes (AWS EKS)
- **Continuous Integration:** GitHub Actions
- **Cloud Provider:** AWS

## Getting Started

### Prerequisites

To run this project, ensure you have the following:

- An AWS account with permissions to create and manage EKS and IAM roles.
- Docker installed on your local machine.
- Access to the GitHub repository with appropriate secrets configured for AWS and Docker Hub.

### Setting Up Secrets

In your GitHub repository, you need to set the following secrets:

- `AWS_ACCESS_KEY_ID`: Your AWS access key ID.
- `AWS_SECRET_ACCESS_KEY`: Your AWS secret access key.
- `DOCKER_USERNAME`: Your Docker Hub username.
- `DOCKER_PASSWORD`: Your Docker Hub password.

## Workflow Overview

The GitHub Actions workflow is defined in the `.github/workflows/deploy.yml` file. It performs the following steps:

1. **Checkout Code:** Retrieves the latest code from the repository.
2. **Set up Docker Buildx:** Configures Docker Buildx for building images.
3. **Set up AWS CLI:** Configures AWS CLI with your AWS credentials.
4. **Log in to Docker Hub:** Authenticates to Docker Hub to push images.
5. **Build and Push Docker Image:** Builds the frontend Docker image and pushes it to Docker Hub.
6. **Configure kubeconfig for AWS EKS:** Updates the kubeconfig for accessing the EKS cluster.
7. **Deploy to Kubernetes:** Applies the Kubernetes manifests located in the `k8s/` directory.

## Deployment

To deploy the application:

1. Ensure your AWS EKS cluster is up and running.
2. Make sure your Kubernetes manifests are correctly configured in the `k8s/` directory.
3. Push changes to the `main` branch of your repository. The GitHub Actions workflow will automatically trigger and deploy your application.'

This project uses a GitHub Actions workflow to automatically build and deploy the application to the Kubernetes cluster whenever changes are pushed to the main branch.
GitHub Actions Workflow

The workflow file is located at .github/workflows/deploy.yml. It performs the following steps:

    Checks out the code.
    Sets up Docker Buildx.
    Configures AWS CLI credentials.
    Logs in to Docker Hub.
    Builds and pushes the Docker image for the frontend.
    Configures the kubeconfig for AWS EKS.
    Deploys the application to Kubernetes.
## Terraform Configuration
The Terraform configuration file sets up the necessary AWS infrastructure, including a VPC, subnets, and an EKS cluster. Below is the provided Terraform code:
    provider "aws" {
  region = "us-east-1"
}

terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 3.0"
    }
  }
}

resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
  tags = {
    Name = "main"
  }
}

resource "aws_internet_gateway" "igw" {
  vpc_id = aws_vpc.main.id
  tags = {
    Name = "igw"
  }
}

resource "aws_subnet" "private_us_east_1a" {
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.0.0/19"
  availability_zone = "us-east-1a"
  tags = {
    Name = "private-us-east-1a"
    "kubernetes.io/role/internal-elb" = "1"
    "kubernetes.io/cluster/demo" = "owned"
  }
}

resource "aws_subnet" "private_us_east_1b" {
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.32.0/19"
  availability_zone = "us-east-1b"
  tags = {
    Name = "private-us-east-1b"
    "kubernetes.io/role/internal-elb" = "1"
    "kubernetes.io/cluster/demo" = "owned"
  }
}

resource "aws_subnet" "public_us_east_1a" {
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.64.0/19"
  availability_zone = "us-east-1a"
  map_public_ip_on_launch = true
  tags = {
    Name = "public-us-east-1a"
    "kubernetes.io/role/elb" = "1"
    "kubernetes.io/cluster/demo" = "owned"
  }
}

resource "aws_subnet" "public_us_east_1b" {
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.96.0/19"
  availability_zone = "us-east-1b"
  map_public_ip_on_launch = true
  tags = {
    Name = "public-us-east-1b"
    "kubernetes.io/role/elb" = "1"
    "kubernetes.io/cluster/demo" = "owned"
  }
}

resource "aws_eip" "nat" {
  tags = {
    Name = "nat"
  }
}

resource "aws_nat_gateway" "nat" {
  allocation_id = aws_eip.nat.id
  subnet_id     = aws_subnet.public_us_east_1a.id
  tags = {
    Name = "nat"
  }
  depends_on = [aws_internet_gateway.igw]
}

resource "aws_route_table" "private" {
  vpc_id = aws_vpc.main.id
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_nat_gateway.nat.id
  }
  tags = {
    Name = "private"
  }
}

resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.igw.id
  }
  tags = {
    Name = "public"
  }
}

resource "aws_route_table_association" "private_us_east_1a" {
  subnet_id      = aws_subnet.private_us_east_1a.id
  route_table_id = aws_route_table.private.id
}

resource "aws_route_table_association" "private_us_east_1b" {
  subnet_id      = aws_subnet.private_us_east_1b.id
  route_table_id = aws_route_table.private.id
}

resource "aws_route_table_association" "public_us_east_1a" {
  subnet_id      = aws_subnet.public_us_east_1a.id
  route_table_id = aws_route_table.public.id
}

resource "aws_route_table_association" "public_us_east_1b" {
  subnet_id      = aws_subnet.public_us_east_1b.id
  route_table_id = aws_route_table.public.id
}

resource "aws_iam_role" "demo" {
  name = "eks-cluster-demo"

  assume_role_policy = <<POLICY
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
POLICY
}

resource "aws_iam_role_policy_attachment" "eks_cluster_AmazonEKSClusterPolicy" {
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKSClusterPolicy"
  role       = aws_iam_role.demo.name
}

resource "aws_eks_cluster" "demo" {
  name     = "demo"
  role_arn = aws_iam_role.demo.arn

  vpc_config {
    subnet_ids = [
      aws_subnet.private_us_east_1a.id,
      aws_subnet.private_us_east_1b.id,
      aws_subnet.public_us_east_1a.id,
      aws_subnet.public_us_east_1b.id
    ]
  }

  depends_on = [aws_iam_role_policy_attachment.eks_cluster_AmazonEKSClusterPolicy]
}

resource "aws_iam_role" "nodes" {
  name = "eks-node-group-nodes"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Principal = {
          Service = "ec2.amazonaws.com"
        }
      }
    ]
  })
}

resource "aws_iam_role_policy_attachment" "nodes_AmazonEKSWorkerNodePolicy" {
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy"
  role       = aws_iam_role.nodes.name
}

resource "aws_iam_role_policy_attachment" "nodes_AmazonEKS_CNI_Policy" {
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy"
  role       = aws_iam_role.nodes.name
}

resource "aws_iam_role_policy_attachment" "nodes_AmazonEC2ContainerRegistryReadOnly" {
  policy_arn = "arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly"
  role       = aws_iam_role.nodes.name
}

resource "aws_eks_node_group" "private_nodes" {
  cluster_name    = aws_eks_cluster.demo.name
  node_group_name = "private-nodes"
  node_role_arn   = aws_iam_role.nodes.arn
  subnet_ids      = [aws_subnet.private_us_east_1a.id, aws_subnet.private_us_east_1b.id]

  scaling_config {
    desired_size = 5
    max_size     = 5
    min_size     = 5
  }

  capacity_type  = "ON_DEMAND"
  instance_types = ["t2.micro"]

  update_config {
    max_unavailable = 1
  }

  labels = {
    role = "general"
  }

  depends_on = [
    aws_iam_role_policy_attachment.nodes_AmazonEKSWorkerNodePolicy,
    aws_iam_role_policy_attachment.nodes_AmazonEKS_CNI_Policy,
    aws_iam_role_policy_attachment.nodes_AmazonEC2ContainerRegistryReadOnly
  ]
}

data "tls_certificate" "eks" {
  url = aws_eks_cluster.demo.identity[0].oidc[0].issuer
}

resource "aws_iam_openid_connect_provider" "eks" {
  client_id_list  = ["sts.amazonaws.com"]
  thumbprint_list = ["9e99a48a9960b14926bb7f3b54024ee01f10049d"]
  url             = aws_eks_cluster.demo.identity[0].oidc[0].issuer
}

data "aws_iam_policy_document" "test_odic_assume_role_policy" {
  statement {
    actions = ["sts:AssumeRoleWithWebIdentity"]
    effect  = "Allow"

    condition {
      test     = "StringEquals"
      variable = "${replace(aws_iam_openid_connect_provider.eks.url, "https://", "")}:sub"
      values   = ["system:serviceaccount:default:aws-test"]
    }

    principals {
      identifiers = [aws_iam_openid_connect_provider.eks.arn]
      type        = "Federated"
    }
  }
}

resource "aws_iam_role" "test_odic" {
  assume_role_policy = data.aws_iam_policy_document.test_odic_assume_role_policy.json
  name               = "test-odic"
}

resource "aws_iam_policy" "test_policy" {
  name   = "test_policy"
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action   = [
          "s3:ListAllMyBuckets",
          "s3:GetBucketLocation"
        ]
        Effect   = "Allow"
        Resource = "arn:aws:s3:::*"
      }
    ]
  })
}

resource "aws_iam_role_policy_attachment" "test_attach" {
  role       = aws_iam_role.test_odic.name
  policy_arn = aws_iam_policy.test_policy.arn
}

data "aws_iam_policy_document" "eks_cluster_autoscaler_assume_role_policy" {
  statement {
    actions = ["sts:AssumeRoleWithWebIdentity"]
    effect = "Allow"

    condition {
      test = "StringEquals"
      variable = "${replace(aws_iam_openid_connect_provider.eks.url, "https://", "")}:sub"
      values = ["system:serviceaccount:kube-system:cluster-autoscaler"]
    }
    principals {
      identifiers = [aws_iam_openid_connect_provider.eks.arn]
      type = "Federated"
    }
  }
}

resource "aws_iam_role" "eks_cluster_autoscaler" {
  assume_role_policy = data.aws_iam_policy_document.eks_cluster_autoscaler_assume_role_policy.json
  name = "eks-cluster-autoscaler"
}

resource "aws_iam_policy" "eks_cluster_autoscaler" {
  name = "eks-cluster-autoscaler"
  policy = jsonencode({
    Statement = [{
      Action = [
        "autoscaling:DescribeAutoscalingGroups",
        "autoscaling:DescribeAutoScalingInstances",
        "autoscaling:DescribeLaunchConfigurations",
        "autoscaling:DescribeTags",
        "autoscaling:SetDesiredCapacity",
        "autoscaling:TerminateInstanceInAutoScalingGroup",
        "ec2:DescribeLaunchTemplateVersions"
      ]
      Effect = "Allow"
      Resource = "*"
    }]
    Version = "2012-10-17"
  })
}

resource "aws_iam_role_policy_attachment" "eks_cluster_autoscaler_attach" {
  role = aws_iam_role.eks_cluster_autoscaler.name
  policy_arn = aws_iam_policy.eks_cluster_autoscaler.arn
}

output "test_policy_arn" {
  value = aws_iam_role.test_odic.arn
}



## Usage

After the deployment is complete, you can access your frontend application via the external IP or LoadBalancer service created in your Kubernetes cluster. Use the following command to check the status of your services:

```bash
kubectl get services

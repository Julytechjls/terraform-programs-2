# Terraform Advanced Practice 

## Introduction

In this advanced practice, you will learn how to use the "file" and "remote-exec" provisioners in Terraform to add and run scripts on an EC2 instance on AWS. You will configure a web server as a practical example.

## Prerequisites

1. AWS account
2. Terraform installed on your local machine
3. AWS CLI configured with your AWS credentials

## Section 1: Installation and Use of the "file" and "remote-exec" Provisioners

### Step 1: Create a Terraform Configuration File

1. Create a directory for your Terraform project and navigate into it:

    ```sh
    mkdir terraform-ec2
    cd terraform-ec2
    touch main.tf
    ```

### Step 2: Configure the AWS Provider

2. In the `main.tf` file, add the AWS provider configuration:

    ```hcl
    provider "aws" {
      region = "us-east-1"  # Change this to your preferred region
    }
    ```

### Step 3: Create an EC2 Instance

3. Add the configuration to create an EC2 instance. Ensure you have an SSH key pair created in AWS to access the instance:

    ```hcl
    resource "aws_instance" "web" {
      ami           = "ami-01b799c439fd5516a"  # Amazon Linux AMI 2
      instance_type = "t2.micro"
      key_name      = "vockey"  # Change this to the name of your SSH key pair

      tags = {
        Name = "WebServer"
      }

      # Defines the Security Group to allow HTTP and SSH traffic
      vpc_security_group_ids = [aws_security_group.web_sg.id]

      provisioner "file" {
        source      = "install_apache.sh"
        destination = "/tmp/install_apache.sh"
      }

      provisioner "remote-exec" {
        inline = [
          "chmod +x /tmp/install_apache.sh",
          "sudo /tmp/install_apache.sh"
        ]
      }

      connection {
        type        = "ssh"
        user        = "ec2-user"
        private_key = file("ssh.pem")  # Path to your private key
        host        = self.public_ip
      }
    }
    ```

### Step 4: Create the Security Group

4. Add settings to create a Security Group that allows HTTP and SSH traffic:

    ```hcl
    resource "aws_security_group" "web_sg" {
      name        = "web_sg"
      description = "Allow HTTP and SSH traffic"

      ingress {
        from_port   = 22
        to_port     = 22
        protocol    = "tcp"
        cidr_blocks = ["0.0.0.0/0"]
      }

      ingress {
        from_port   = 80
        to_port     = 80
        protocol    = "tcp"
        cidr_blocks = ["0.0.0.0/0"]
      }

      egress {
        from_port   = 0
        to_port     = 0
        protocol    = "-1"
        cidr_blocks = ["0.0.0.0/0"]
      }
    }
    ```

### Step 5: Create the Apache Installation Script

5. Create a file called `install_apache.sh` in the same directory:

    ```sh
    touch install_apache.sh
    ```

6. Add the following content to the `install_apache.sh` file to install and configure Apache:

    ```bash
    #!/bin/bash

    # Update the packages and install Apache
    sudo yum update -y
    sudo yum install -y httpd

    # Start Apache and enable it to start on every system reboot
    sudo systemctl start httpd
    sudo systemctl enable httpd

    # Create an example web page
    echo "<html><h1>Hello from Terraform!</h1></html>" | sudo tee /var/www/html/index.html
    ```

### Step 6: Obtain the Private SSH Key

7. Obtain the private SSH key from the lab and paste the contents into a file called `ssh.pem` in the root path of the Terraform project.

### Step 7: Initialize and Apply Terraform Configuration

8. Initialize your Terraform project and apply the settings:

    ```sh
    terraform init
    terraform apply
    ```

9. Confirm the application when prompted (type `yes` and press Enter).

10. Once the application completes, you will get the public IP address of the EC2 instance. You can verify that Apache is installed and running by accessing this IP address in your browser. Create an output in Terraform that delivers the public IP of the instance:

    ```hcl
    output "instance_public_ip" {
      value = aws_instance.web.public_ip
    }
    ```

## PRACTICE 2: Creating a VPC with Conditional Subnets

### Add the VPC Resource

1. Add the VPC resource:

    ```hcl
    resource "aws_vpc" "main" {
      cidr_block = "10.0.0.0/16"
    }
    ```

### Add Subnets Using Conditional Expressions

2. Add subnets using conditional expressions:

    ```hcl
    variable "environment" {
      description = "The environment to deploy to"
      type        = string
      default     = "dev"
    }

    resource "aws_subnet" "subnet" {
      count = var.environment == "prod" ? 2 : 1
      vpc_id = aws_vpc.main.id
      cidr_block = var.environment == "prod" ? element(["10.0.1.0/24", "10.0.2.0/24"], count.index) : "10.0.1.0/24"
      availability_zone = element(data.aws_availability_zones.available.names, count.index)
    }

    data "aws_availability_zones" "available" {
      state = "available"
    }
    ```

### Application of Operators and Function Calls

3. Create a security group resource with operator-based rules:

    ```hcl
    resource "aws_security_group" "allow_tls" {
      name_prefix = "allow_tls_"
      vpc_id      = aws_vpc.main.id

      ingress {
        description = "TLS from VPC"
        from_port   = 443
        to_port     = 443
        protocol    = "tcp"
        cidr_blocks = ["0.0.0.0/0"]
      }

      egress {
        from_port   = 0
        to_port     = 0
        protocol    = "-1"
        cidr_blocks = ["0.0.0.0/0"]
      }
    }
    ```

### Use Functions and Operators in Configurations

4. Use functions and operators in configurations:

    ```hcl
    resource "aws_instance" "app" {
      ami           = "ami-01b799c439fd5516a"  # Amazon Linux 2 AMI
      instance_type = var.environment == "prod" ? "t2.medium" : "t2.micro"
      subnet_id     = element(aws_subnet.subnet[*].id, 0)
      vpc_security_group_ids = [aws_security_group.allow_tls.id]

      tags = {
        Name = "MyAppInstance"
      }

      root_block_device {
        volume_size = var.environment == "prod" ? 50 : 20
        volume_type = "gp2"
      }
    }
    ```

### Using For and Splat Expressions

5. Create a list of subnet names using `for`:

    ```hcl
    locals {
      subnet_names = [for i in aws_subnet.subnet : "subnet-${i.availability_zone}"]
    }

    output "subnet_names" {
      value = local.subnet_names
    }
    ```

6. Use splat to get the subnet IDs:

    ```hcl
    output "subnet_ids" {
      value = aws_subnet.subnet[*].id
    }
    ```

## Running Terraform Configuration

### Initialize Terraform

1. Initialize Terraform:

    ```sh
    terraform init
    ```

### Review the Execution Plan

2. Review the execution plan:

    ```sh
    terraform plan
    ```

### Apply the Plan

3. Apply the plan:

    ```sh
    terraform apply
    ```

Confirm the application when prompted (type `yes` and press Enter).

---

Customize paths and variable values as needed for your specific environment.

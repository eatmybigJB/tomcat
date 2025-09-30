```python
packer {
  required_plugins {
    amazon = {
      source  = "github.com/hashicorp/amazon"
      version = ">= 1.0.0"
    }
  }
}

variable "aws_region" {
  default = "us-east-1"
}

variable "vpc_id" {
  default = "vpc-xxxxxxxx" # 你的VPC ID
}

variable "subnet_id" {
  default = "subnet-xxxxxxxx" # 你的子网ID
}

variable "security_group_id" {
  default = "sg-xxxxxxxx" # 你的安全组ID
}

source "amazon-ebs" "ubuntu" {
  region                  = var.aws_region
  instance_type           = "t2.micro"
  ami_name                = "packer-ubuntu-{{timestamp}}"

  # Ubuntu 官方 AMI 过滤
  source_ami_filter {
    filters = {
      name                = "ubuntu/images/hvm-ssd/ubuntu-focal-20.04-amd64-server-*"
      root-device-type    = "ebs"
      virtualization-type = "hvm"
    }
    owners      = ["099720109477"] # Canonical
    most_recent = true
  }

  ssh_username            = "ubuntu"

  # VPC 配置
  vpc_id                  = var.vpc_id
  subnet_id               = var.subnet_id
  associate_public_ip_address = true
  security_group_id       = var.security_group_id
}

build {
  name    = "ubuntu-ami"
  sources = ["source.amazon-ebs.ubuntu"]

  provisioner "shell" {
    inline = [
      "echo 'Hello from Packer build with VPC config!' > /home/ubuntu/packer_test.txt",
      "sudo apt-get update -y && sudo apt-get install -y htop"
    ]
  }
}

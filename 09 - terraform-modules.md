## Objective
- Understand module creation
- Structure a reusable module
- Use input variables and outputs
- Call the module from a root configuration

```yaml
terraform-module-lab/
├── main.tf
├── variables.tf
├── outputs.tf
├── terraform.tfvars
├── modules/
│   └── ec2_instance/
│       ├── main.tf
│       ├── variables.tf
│       └── outputs.tf
```


## Step 2: Create EC2 Module (modules/ec2_instance/)
### modules/ec2_instance/main.tf
```yaml
resource "aws_instance" "this" {
  ami                    = var.ami
  instance_type          = var.instance_type
  subnet_id              = var.subnet_id
  vpc_security_group_ids = var.security_groups
  tags = {
    Name = var.name
  }
}
```

### modules/ec2_instance/variables.tf
```yaml
variable "ami" {
  description = "AMI ID for the EC2 instance"
  type        = string
}

variable "instance_type" {
  description = "EC2 instance type"
  type        = string
}

variable "subnet_id" {
  description = "Subnet ID"
  type        = string
}

variable "security_groups" {
  description = "List of security group IDs"
  type        = list(string)
}

variable "name" {
  description = "Name tag for the instance"
  type        = string
}
```

### modules/ec2_instance/outputs.tf
```yaml
output "instance_id" {
  value = aws_instance.this.id
}

output "public_ip" {
  value = aws_instance.this.public_ip
}
```

## Step 3: Root Configuration
### main.tf
```yaml
provider "aws" {
  region = "us-east-1"
}

module "web_server" {
  source          = "./modules/ec2_instance"
  ami             = var.ami
  instance_type   = var.instance_type
  subnet_id       = var.subnet_id
  security_groups = var.security_groups
  name            = "WebServer"
}
```
### variables.tf
```yaml
variable "ami" {
  type        = string
  description = "AMI to use for the instance"
}

variable "instance_type" {
  type        = string
  description = "Type of EC2 instance"
}

variable "subnet_id" {
  type        = string
  description = "Subnet ID to launch the instance in"
}

variable "security_groups" {
  type        = list(string)
  description = "List of security group IDs"
}
```
### terraform.tfvars
```yaml
ami             = "ami-0c55b159cbfafe1f0"
instance_type   = "t2.micro"
subnet_id       = "subnet-xxxxxxx"
security_groups = ["sg-xxxxxxx"]
```

### outputs.tf
```yaml
output "ec2_instance_id" {
  value = module.web_server.instance_id
}

output "ec2_public_ip" {
  value = module.web_server.public_ip
}
```

## Step 4: Initialize and Apply
```yaml
terraform init
terraform plan
terraform apply
```

## Step 5: Validate Outputs
Check if:
- An EC2 instance is launched
- Outputs (public IP, instance ID) are printed
- You can SSH or curl the instance (if security group allows)

## Step 6: Destroy Resources
```yaml
terraform destroy
```

### What You Learned
- Creating reusable modules
- Passing variables and reading outputs
- Organizing Terraform code cleanly
- Promoting modular, DRY infrastructure


-----


🔁 Extended Lab: Demonstrating Reusable Terraform Modules
🧱 Updated Directory Structure
```bash
terraform-module-lab/
├── environments/
│   ├── dev/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── terraform.tfvars
│   │   └── backend.tf
│   └── prod/
│       ├── main.tf
│       ├── variables.tf
│       ├── terraform.tfvars
│       └── backend.tf
├── modules/
│   ├── ec2_instance/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   └── vpc/
│       ├── main.tf
│       ├── variables.tf
│       └── outputs.tf
```

## Step 1: Add a Reusable vpc Module
### modules/vpc/main.tf
```yaml
resource "aws_vpc" "main" {
  cidr_block = var.cidr_block
  tags = {
    Name = var.name
  }
}

resource "aws_subnet" "main" {
  vpc_id            = aws_vpc.main.id
  cidr_block        = var.subnet_cidr
  availability_zone = var.availability_zone
  tags = {
    Name = "${var.name}-subnet"
  }
}
```

### modules/vpc/variables.tf
```yaml
variable "cidr_block" {
  type        = string
  description = "CIDR block for the VPC"
}

variable "subnet_cidr" {
  type        = string
  description = "CIDR block for the subnet"
}

variable "availability_zone" {
  type        = string
  description = "AZ for subnet"
}

variable "name" {
  type        = string
  description = "Name tag"
}
```

### modules/vpc/outputs.tf
```yaml
output "vpc_id" {
  value = aws_vpc.main.id
}

output "subnet_id" {
  value = aws_subnet.main.id
}
```

## Step 2: Dev Environment – Use Modules Together
### environments/dev/backend.tf
```yaml
terraform {
  backend "local" {
    path = "terraform.tfstate"
  }
}
```

### environments/dev/variables.tf
```yaml
variable "ami" {}
variable "instance_type" {}
variable "availability_zone" {}
```

### environments/dev/terraform.tfvars
```yaml
ami                = "ami-0c55b159cbfafe1f0"
instance_type      = "t2.micro"
availability_zone  = "us-east-1a"
```

### environments/dev/main.tf
```yaml
provider "aws" {
  region = "us-east-1"
}

module "vpc" {
  source            = "../../modules/vpc"
  cidr_block        = "10.0.0.0/16"
  subnet_cidr       = "10.0.1.0/24"
  availability_zone = var.availability_zone
  name              = "dev-vpc"
}

module "web" {
  source          = "../../modules/ec2_instance"
  ami             = var.ami
  instance_type   = var.instance_type
  subnet_id       = module.vpc.subnet_id
  security_groups = []  # Add SG if needed
  name            = "dev-web"
}

module "bastion" {
  source          = "../../modules/ec2_instance"
  ami             = var.ami
  instance_type   = var.instance_type
  subnet_id       = module.vpc.subnet_id
  security_groups = []
  name            = "dev-bastion"
}
```

## Step 3: Run and Validate (for Dev)
```yaml
cd environments/dev
terraform init
terraform apply
```

**You should see:**
- A VPC and subnet created
- Two EC2 instances: one web, one bastion

## Step 4: Create Prod Environment
**Duplicate the dev/ folder as prod/, adjust:**

- CIDR ranges
- Tags
- Backend (to use S3, for example)

**This showcases:**
- Different configs using same modules
- Centralized reusable logic


### Key Concepts Demonstrated
- Reusability: Modules used multiple times in one environment (web, bastion)
- Multi-env setup: dev and prod folders with shared modules
- Backend config: backend.tf with local/S3 backends
- Separation of concerns: Modules in modules/, env-specific files in environments/
- Outputs chaining: Subnet ID from VPC module used in EC2 module

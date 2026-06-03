# Complete setup for VPC:

1. VPC
2. Public + Private Subnets
3. Internet Gateway (IGW)
4. NAT Gateway
5. Route Tables
6. Module-based structure

### Final Project Structure
```
terraform-project/
│
├── main.tf
├── variables.tf
├── outputs.tf
├── provider.tf
├── terraform.tfvars
│
└── modules/
    ├── vpc/
    │   ├── main.tf
    │   ├── variables.tf
    │   └── outputs.tf
    │
    ├── subnet/
    │   ├── main.tf
    │   ├── variables.tf
    │   └── outputs.tf
    │
    ├── igw/
    │   ├── main.tf
    │   ├── variables.tf
    │   └── outputs.tf
    │
    ├── nat/
    │   ├── main.tf
    │   ├── variables.tf
    │   └── outputs.tf
    │
    └── route-table/
        ├── main.tf
        ├── variables.tf
        └── outputs.tf
```
## STEP 1 — Create Project Structure
```bash
mkdir terraform-project
cd terraform-project

mkdir -p modules/{vpc,subnet,igw,nat,route-table}

touch main.tf variables.tf outputs.tf provider.tf terraform.tfvars

touch modules/vpc/{main.tf,variables.tf,outputs.tf}

touch modules/subnet/{main.tf,variables.tf,outputs.tf}

touch modules/igw/{main.tf,variables.tf,outputs.tf}

touch modules/nat/{main.tf,variables.tf,outputs.tf}

touch modules/route-table/{main.tf,variables.tf,outputs.tf}
```

## STEP 2 — Root Provider File

provider.tf
```bash
provider "aws" {
  region = var.aws_region
}

STEP 3 — Root Variables
variables.tf

variable "aws_region" {
  default = "us-east-1"
}

variable "vpc_cidr" {
  default = "10.0.0.0/16"
}

variable "public_subnet_cidr" {
  default = "10.0.1.0/24"
}

variable "private_subnet_cidr" {
  default = "10.0.2.0/24"
}

variable "availability_zone" {
  default = "us-east-1a"
}
```

## STEP 4 — Root Main File
main.tf
```bash
# VPC MODULE
module "vpc" {
  source   = "./modules/vpc"
  vpc_cidr = var.vpc_cidr
}

# PUBLIC SUBNET
module "public_subnet" {
  source            = "./modules/subnet"
  vpc_id            = module.vpc.vpc_id
  subnet_cidr       = var.public_subnet_cidr
  availability_zone = var.availability_zone
  subnet_name       = "PublicSubnet"
}

# PRIVATE SUBNET
module "private_subnet" {
  source            = "./modules/subnet"
  vpc_id            = module.vpc.vpc_id
  subnet_cidr       = var.private_subnet_cidr
  availability_zone = var.availability_zone
  subnet_name       = "PrivateSubnet"
}

# INTERNET GATEWAY
module "igw" {
  source = "./modules/igw"
  vpc_id = module.vpc.vpc_id
}

# NAT GATEWAY
module "nat" {
  source          = "./modules/nat"
  public_subnet_id = module.public_subnet.subnet_id
}

# PUBLIC ROUTE TABLE
module "public_route_table" {
  source     = "./modules/route-table"
  vpc_id     = module.vpc.vpc_id
  subnet_id  = module.public_subnet.subnet_id
  gateway_id = module.igw.igw_id
  route_type = "igw"
}

# PRIVATE ROUTE TABLE
module "private_route_table" {
  source     = "./modules/route-table"
  vpc_id     = module.vpc.vpc_id
  subnet_id  = module.private_subnet.subnet_id
  gateway_id = module.nat.nat_gateway_id
  route_type = "nat"
}
```

## STEP 5 — VPC MODULE
modules/vpc/variables.tf
```bash
variable "vpc_cidr" {}
modules/vpc/main.tf
resource "aws_vpc" "this" {
  cidr_block = var.vpc_cidr

  tags = {
    Name = "MyVPC"
  }
}
modules/vpc/outputs.tf
output "vpc_id" {
  value = aws_vpc.this.id
}
```

## STEP 6 — SUBNET MODULE
modules/subnet/variables.tf
```bash
variable "vpc_id" {}

variable "subnet_cidr" {}

variable "availability_zone" {}

variable "subnet_name" {}
modules/subnet/main.tf
resource "aws_subnet" "this" {
  vpc_id                  = var.vpc_id
  cidr_block              = var.subnet_cidr
  availability_zone       = var.availability_zone
  map_public_ip_on_launch = true

  tags = {
    Name = var.subnet_name
  }
}
modules/subnet/outputs.tf
output "subnet_id" {
  value = aws_subnet.this.id
}
```

## STEP 7 — IGW MODULE
modules/igw/variables.tf
```bash
variable "vpc_id" {}
modules/igw/main.tf
resource "aws_internet_gateway" "this" {
  vpc_id = var.vpc_id

  tags = {
    Name = "MyIGW"
  }
}
modules/igw/outputs.tf
output "igw_id" {
  value = aws_internet_gateway.this.id
}
```

## STEP 8 — NAT MODULE
modules/nat/variables.tf
```bash
variable "public_subnet_id" {}
modules/nat/main.tf
resource "aws_eip" "this" {
  domain = "vpc"
}

resource "aws_nat_gateway" "this" {
  allocation_id = aws_eip.this.id
  subnet_id     = var.public_subnet_id

  tags = {
    Name = "MyNAT"
  }

  depends_on = [aws_eip.this]
}
modules/nat/outputs.tf
output "nat_gateway_id" {
  value = aws_nat_gateway.this.id
}
```

## STEP 9 — ROUTE TABLE MODULE
modules/route-table/variables.tf
```bash
variable "vpc_id" {}

variable "subnet_id" {}

variable "gateway_id" {}

variable "route_type" {}
modules/route-table/main.tf
resource "aws_route_table" "this" {
  vpc_id = var.vpc_id

  dynamic "route" {
    for_each = [1]

    content {
      cidr_block = "0.0.0.0/0"

      gateway_id     = var.route_type == "igw" ? var.gateway_id : null
      nat_gateway_id = var.route_type == "nat" ? var.gateway_id : null
    }
  }

  tags = {
    Name = "${var.route_type}-route-table"
  }
}

resource "aws_route_table_association" "this" {
  subnet_id      = var.subnet_id
  route_table_id = aws_route_table.this.id
}
modules/route-table/outputs.tf
output "route_table_id" {
  value = aws_route_table.this.id
}
```

## STEP 10 — Root Outputs
outputs.tf
```bash
output "vpc_id" {
  value = module.vpc.vpc_id
}

output "public_subnet_id" {
  value = module.public_subnet.subnet_id
}

output "private_subnet_id" {
  value = module.private_subnet.subnet_id
}

output "igw_id" {
  value = module.igw.igw_id
}

output "nat_gateway_id" {
  value = module.nat.nat_gateway_id
}
```

## STEP 11 — Terraform Commands

Initialize
```bash
terraform init
```
Validate
```bash
terraform validate
```
Plan
```bash
terraform plan
```
Apply
```bash
terraform apply
```

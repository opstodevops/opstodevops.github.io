---
layout: post
title:  "Using AWS S3 Bucket and DynamoDB for Terraform backend"
date:   2020-07-08
categories: [blog]
tags: [aws, hashicorp, terraform, s3, dynamodb]
excerpt_separator: <!--more-->
---
Terraform state management is always a topic to discuss when comparing Terraform vs Terraform Enterprise.
In this blog, I will cover a simple method of remote state management using S3 and DynamoDB table.
It is known that Terraform learns about changes in infrastructure using its state file which is named terraform.tfstate & can be found in .terraform (hidden) folder.
There is `NEVER` a good reason to touch this file but a quick glance will show current state of the infrastructure.

<!--more-->

```bash
OpsToDevOps : ls -al
total 20
drwxrwxr-x 3 ubuntu ubuntu 4096 Jul  7 21:22 .
drwxr-xr-x 5 ubuntu ubuntu 4096 Jul  7 19:39 ..
drwxrwxr-x 4 ubuntu ubuntu 4096 Jul  7 21:18 .terraform
-rw-rw-r-- 1 ubuntu ubuntu  126 Jul  7 19:45 backend.tf
-rw-rw-r-- 1 ubuntu ubuntu 1401 Jul  7 21:06 main.tf
OpsToDevOps : cd .terraform/
OpsToDevOps : ls -al
total 20
drwxrwxr-x 4 ubuntu ubuntu 4096 Jul  7 21:18 .
drwxrwxr-x 3 ubuntu ubuntu 4096 Jul  7 21:22 ..
drwxrwxr-x 3 ubuntu ubuntu 4096 Jul  7 21:18 modules
drwxr-xr-x 3 ubuntu ubuntu 4096 Jul  7 21:18 plugins
-rw-rw-r-- 1 ubuntu ubuntu 1534 Jul  7 21:18 terraform.tfstate
```

If the state is stored on one developer’s workstation which is mostly the case then others cannot use it in their workflow or make changes to infrastructure without breaking something.
The reason & you guessed it right is local storage of terraform.tfstate file.

In this blog, I will show a method to use AWS S3 as a backend to remotely store terraform state so that all developers can have access to current state of infrastructure. 
I will also show the benefit of remote state locking using DynamoDB to avoid conflicts when 2 developers decide to work on same infrastructure state file.

Here is my directory structure; I have 2 folders suitably named after the objective. In `remote-state`, I have a file, `main.tf` which will create S3 Bucket and DynamoDB table for us. 

![tf-backend-dir-struct_1a](/assets/tf-backend-dir-struct.PNG)

NOTE --> I am using the `rand` plugin for adding randomness to names of my S3 bucket and DynamoDB table.

```go{% raw %}
##########
# VARIABLES
##########

variable "region" {
  type    = string
  default = "us-east-1"
}

#Bucket variables
variable "aws_bucket_prefix" {
  type    = string
  default = "ops2dev-tfstatebucket"
}

variable "aws_dynamodb_table" {
  type    = string
  default = "ops2dev-tfstatelock"
}

############
# PROVIDERS
############

provider "aws" {
  access_key = "YOUR ACCESS KEY"
  secret_key = "YOUR SECRET ACCESS KEY"
  region     = "us-east-1"
}

############
# RESOURCES
############

resource "random_integer" "rand" {
  min = 10000
  max = 99999
}

locals {

  dynamodb_table_name = "${var.aws_dynamodb_table}-${random_integer.rand.result}"
  bucket_name         = "${var.aws_bucket_prefix}-${random_integer.rand.result}"
}

resource "aws_dynamodb_table" "terraform_statelock" {
  name           = local.dynamodb_table_name
  read_capacity  = 20
  write_capacity = 20
  hash_key       = "LockID"

  attribute {
    name = "LockID"
    type = "S" # String
  }
}

resource "aws_s3_bucket" "state_bucket" {
  bucket        = local.bucket_name
  acl           = "private"
  force_destroy = true

  versioning {
    enabled = true
  }

}

resource "aws_iam_group" "bucket_full_access" {

  name = "${local.bucket_name}-full-access"

}

resource "aws_iam_user" "user_opstodevops" {
  name = "opstodevops"
}

# Add user to the group

resource "aws_iam_group_membership" "full_access" {
  name = "${local.bucket_name}-full-access"

  users = ["${aws_iam_user.user_opstodevops.name}"]

  group = aws_iam_group.bucket_full_access.name
}

resource "aws_iam_group_policy" "full_access" {
  name  = "${local.bucket_name}-full-access"
  group = aws_iam_group.bucket_full_access.id

  policy = <<EOF
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "s3:*",
            "Resource": [
                "arn:aws:s3:::${local.bucket_name}",
                "arn:aws:s3:::${local.bucket_name}/*"
            ]
        },
                {
            "Effect": "Allow",
            "Action": ["dynamodb:*"],
            "Resource": [
                "${aws_dynamodb_table.terraform_statelock.arn}"
            ]
        }
   ]
}
EOF
}

##########
# OUTPUT
##########

output "s3_bucket" {
  value = aws_s3_bucket.state_bucket.bucket
}

output "dynamodb_statelock" {
  value = aws_dynamodb_table.terraform_statelock.name
}
``` {% endraw %}

I am going to `PLAN` the configuration and see what’s going to be built.

```bash
OpsToDevOps $ terraform plan -out remote-state.tf
Refreshing Terraform state in-memory prior to plan...
The refreshed state will be used to calculate this plan, but will not be
persisted to local or remote state storage.


------------------------------------------------------------------------

An execution plan has been generated and is shown below.
Resource actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  # aws_dynamodb_table.terraform_statelock will be created
  + resource "aws_dynamodb_table" "terraform_statelock" {
      + arn              = (known after apply)
      + billing_mode     = "PROVISIONED"
      + hash_key         = "LockID"
      + id               = (known after apply)
      + name             = (known after apply)
      + read_capacity    = 20
      + stream_arn       = (known after apply)
      + stream_label     = (known after apply)
      + stream_view_type = (known after apply)
      + write_capacity   = 20

      + attribute {
          + name = "LockID"
          + type = "S"
        }

      + point_in_time_recovery {
          + enabled = (known after apply)
        }

      + server_side_encryption {
          + enabled     = (known after apply)
          + kms_key_arn = (known after apply)
        }
    }

  # aws_iam_group.bucket_full_access will be created
  + resource "aws_iam_group" "bucket_full_access" {
      + arn       = (known after apply)
      + id        = (known after apply)
      + name      = (known after apply)
      + path      = "/"
      + unique_id = (known after apply)
    }

  # aws_iam_group_membership.full_access will be created
  + resource "aws_iam_group_membership" "full_access" {
      + group = (known after apply)
      + id    = (known after apply)
      + name  = (known after apply)
      + users = [
          + "opstodevops",
        ]
    }

  # aws_iam_group_policy.full_access will be created
  + resource "aws_iam_group_policy" "full_access" {
      + group  = (known after apply)
      + id     = (known after apply)
      + name   = (known after apply)
      + policy = (known after apply)
    }

  # aws_iam_user.user_opstodevops will be created
  + resource "aws_iam_user" "user_opstodevops" {
      + arn           = (known after apply)
      + force_destroy = false
      + id            = (known after apply)
      + name          = "opstodevops"
      + path          = "/"
      + unique_id     = (known after apply)
    }

  # aws_s3_bucket.state_bucket will be created
  + resource "aws_s3_bucket" "state_bucket" {
      + acceleration_status         = (known after apply)
      + acl                         = "private"
      + arn                         = (known after apply)
      + bucket                      = (known after apply)
      + bucket_domain_name          = (known after apply)
      + bucket_regional_domain_name = (known after apply)
      + force_destroy               = true
      + hosted_zone_id              = (known after apply)
      + id                          = (known after apply)
      + region                      = (known after apply)
      + request_payer               = (known after apply)
      + website_domain              = (known after apply)
      + website_endpoint            = (known after apply)

      + versioning {
          + enabled    = true
          + mfa_delete = false
        }
    }

  # random_integer.rand will be created
  + resource "random_integer" "rand" {
      + id     = (known after apply)
      + max    = 99999
      + min    = 10000
      + result = (known after apply)
    }

Plan: 7 to add, 0 to change, 0 to destroy.

------------------------------------------------------------------------
This plan was saved to: remote-state.tf

To perform exactly these actions, run the following command to apply:
    terraform apply "remote-state.tf"
```
Ok, everything seems to be in order, let’s `APPLY` my plan to create the S3 bucket and DynamoDB table which I will use to initialize Terraform in next step. Take note of the `Outputs` at the end, which will come handy in next step.

```bash
OpsToDevOps $ terraform apply "remote-state.tf"
random_integer.rand: Creating...
random_integer.rand: Creation complete after 0s [id=33444]
aws_iam_group.bucket_read_only: Creating...
aws_dynamodb_table.terraform_statelock: Creating...
aws_iam_group.bucket_full_access: Creating...
aws_s3_bucket.state_bucket: Creating...
aws_iam_group.bucket_read_only: Creation complete after 1s [id=ops2dev-tfstatebucket-33444-read-only]
aws_iam_group.bucket_full_access: Creation complete after 1s [id=ops2dev-tfstatebucket-33444-full-access]
aws_iam_group_membership.read_only: Creating...
aws_iam_group_policy.read_only: Creating...
aws_iam_group_membership.full_access: Creating...
aws_iam_group_membership.full_access: Creation complete after 0s [id=ops2dev-tfstatebucket-33444-full-access]
aws_iam_group_membership.read_only: Creation complete after 0s [id=ops2dev-tfstatebucket-33444-read-only]
aws_iam_group_policy.read_only: Creation complete after 0s [id=ops2dev-tfstatebucket-33444-read-only:ops2dev-tfstatebucket-33444-read-only]
aws_s3_bucket.state_bucket: Creation complete after 1s [id=ops2dev-tfstatebucket-33444]
aws_dynamodb_table.terraform_statelock: Creation complete after 7s [id=ops2dev-tfstatelock-33444]
aws_iam_group_policy.full_access: Creating...
aws_iam_group_policy.full_access: Creation complete after 0s [id=ops2dev-tfstatebucket-33444-full-access:ops2dev-tfstatebucket-33444-full-access]

Apply complete! Resources: 7 added, 0 changed, 0 destroyed.

The state of your infrastructure has been saved to the path
below. This state is required to modify and destroy your
infrastructure, so keep it safe. To inspect the complete state
use the `terraform show` command.

State path: terraform.tfstate

Outputs:

dynamodb_statelock = ops2dev-tfstatelock-33444
s3_bucket = ops2dev-tfstatebucket-33444
```
Now navigating to my `dev-state` directory which has the file for creating my infrastructure and here I am simply creating a VPC and some public and private subnets.

```go{% raw %}
############
# VARIABLES
############

variable "region" {
  type = string
  default = "us-east-1"
}

variable "vpc_cidr_range" {
  type = string
  default = "10.0.0.0/16"
}

variable "public_subnets" {
  type = list(string)
  default = ["10.0.0.0/24", "10.0.1.0/24"]
}

variable "database_subnets" {
  type = list(string)
  default = ["10.0.8.0/24", "10.0.9.0/24"]
}

############
# PROVIDERS
############

provider "aws" {
  access_key = "YOUR ACCESS KEY"
  secret_key = "YOUR SECRET ACCESS KEY"
  region     = "us-east-1"
}

###############
# DATA SOURCES
###############

data "aws_availability_zones" "azs" {}


############
# RESOURCES
############


module "vpc" {
  source = "terraform-aws-modules/vpc/aws"
  version = "2.33.0"

  name = "dev-vpc"
  cidr = var.vpc_cidr_range

  azs = slice(data.aws_availability_zones.azs.names, 0, 2) # Grabbing 2 AZs from the list of AZs
  
  # Public Subnets
  public_subnets = var.public_subnets

  # Database Subnets
  database_subnets = var.database_subnets
  database_subnet_group_tags = {
    subnet_type = "database"
  }

  tags = {
    Environment = "dev"
    
    }

}


############
# OUTPUT
############


output "vpc_id" {
  value = "module.vpc.vpc.id"
}

output "db_subnet_group" {
  value = "module.vpc.database_subnet_group"
}

output "public_subnets" {
  value = "module.vpc.public_subnets"
}
```{% endraw %}

Here I am using the DynamoDB table and S3 bucket to initialize Terraform. The `VALUES` for each were noted in earlier step.

```bash
OpsToDevOps : terraform init --backend-config="dynamodb_table=ops2dev-tfstatelock-33444" --backend-config="bucket=ops2dev-tfstatebucket-33444"
Initializing modules...
Downloading terraform-aws-modules/vpc/aws 2.33.0 for vpc...
- vpc in .terraform/modules/vpc/terraform-aws-vpc-2.33.0

Initializing the backend...

Successfully configured the backend "s3"! Terraform will automatically
use this backend unless the backend configuration changes.

Initializing provider plugins...
- Checking for available provider plugins...
- Downloading plugin for provider "aws" (hashicorp/aws) 2.69.0...

Terraform has been successfully initialized!

You may now begin working with Terraform. Try running "terraform plan" to see
any changes that are required for your infrastructure. All Terraform commands
should now work.

If you ever set or change modules or backend configuration for Terraform,
rerun this command to reinitialize your working directory. If you forget, other
commands will detect it and remind you to do so if necessary.
```

Once the backend is initialized, now I am going to run the `PLAN` and then subsequently `APPLY`.

```bash
OpsToDevOps : terraform plan -out dev-state.tf                                                                                                                                                                 
Refreshing Terraform state in-memory prior to plan...
The refreshed state will be used to calculate this plan, but will not be
persisted to local or remote state storage.

data.aws_availability_zones.azs: Refreshing state...

------------------------------------------------------------------------

An execution plan has been generated and is shown below.
Resource actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  # module.vpc.aws_db_subnet_group.database[0] will be created
  + resource "aws_db_subnet_group" "database" {
      + arn         = (known after apply)
      + description = "Database subnet group for dev-vpc"
      + id          = (known after apply)
      + name        = "dev-vpc"
      + name_prefix = (known after apply)
      + subnet_ids  = (known after apply)
      + tags        = {
          + "Environment" = "dev"
          + "Name"        = "dev-vpc"
          + "Region"      = "east"          
          + "subnet_type" = "database"
        }
    }

  # module.vpc.aws_internet_gateway.this[0] will be created
  + resource "aws_internet_gateway" "this" {
      + arn      = (known after apply)
      + id       = (known after apply)
      + owner_id = (known after apply)
      + tags     = {
          + "Environment" = "dev"
          + "Name"        = "dev-vpc"
          + "Region"      = "east"       
        }
      + vpc_id   = (known after apply)
    }

  # module.vpc.aws_route.public_internet_gateway[0] will be created
  + resource "aws_route" "public_internet_gateway" {
      + destination_cidr_block     = "0.0.0.0/0"
      + destination_prefix_list_id = (known after apply)
      + egress_only_gateway_id     = (known after apply)
      + gateway_id                 = (known after apply)
      + id                         = (known after apply)
      + instance_id                = (known after apply)
      + instance_owner_id          = (known after apply)
      + nat_gateway_id             = (known after apply)
      + network_interface_id       = (known after apply)
      + origin                     = (known after apply)
      + route_table_id             = (known after apply)
      + state                      = (known after apply)
      + timeouts {
          + create = "5m"
        }
    }

  # module.vpc.aws_route_table.private[0] will be created
  + resource "aws_route_table" "private" {
      + id               = (known after apply)
      + owner_id         = (known after apply)
      + propagating_vgws = (known after apply)
      + route            = (known after apply)
      + tags             = {
          + "Environment" = "dev"
          + "Name"        = "dev-vpc-private-us-east-1a"
          + "Region"      = "east"          
        }
      + vpc_id           = (known after apply)
    }

  # module.vpc.aws_route_table.private[1] will be created
  + resource "aws_route_table" "private" {
      + id               = (known after apply)
      + owner_id         = (known after apply)
      + propagating_vgws = (known after apply)
      + route            = (known after apply)
      + tags             = {
          + "Environment" = "dev"
          + "Name"        = "dev-vpc-private-us-east-1b"
          + "Region"      = "east"          
        }
      + vpc_id           = (known after apply)
    }

  # module.vpc.aws_route_table.public[0] will be created
  + resource "aws_route_table" "public" {
      + id               = (known after apply)
      + owner_id         = (known after apply)
      + propagating_vgws = (known after apply)
      + route            = (known after apply)
      + tags             = {
          + "Environment" = "dev"
          + "Name"        = "dev-vpc-public"
          + "Region"      = "east"          
        }
      + vpc_id           = (known after apply)
    }

  # module.vpc.aws_route_table_association.database[0] will be created
  + resource "aws_route_table_association" "database" {
      + id             = (known after apply)
      + route_table_id = (known after apply)
      + subnet_id      = (known after apply)
    }

  # module.vpc.aws_route_table_association.database[1] will be created
  + resource "aws_route_table_association" "database" {
      + id             = (known after apply)
      + route_table_id = (known after apply)
      + subnet_id      = (known after apply)
    }

  # module.vpc.aws_route_table_association.public[0] will be created
  + resource "aws_route_table_association" "public" {
      + id             = (known after apply)
      + route_table_id = (known after apply)
      + subnet_id      = (known after apply)
    }

  # module.vpc.aws_route_table_association.public[1] will be created
  + resource "aws_route_table_association" "public" {
      + id             = (known after apply)
      + route_table_id = (known after apply)
      + subnet_id      = (known after apply)
    }

  # module.vpc.aws_subnet.database[0] will be created
  + resource "aws_subnet" "database" {
      + arn                             = (known after apply)
      + assign_ipv6_address_on_creation = false
      + availability_zone               = "us-east-1a"
      + availability_zone_id            = (known after apply)
      + cidr_block                      = "10.0.8.0/24"
      + id                              = (known after apply)
      + ipv6_cidr_block                 = (known after apply)
      + ipv6_cidr_block_association_id  = (known after apply)
      + map_public_ip_on_launch         = false
      + owner_id                        = (known after apply)
      + tags                            = {
          + "Environment" = "dev"
          + "Name"        = "dev-vpc-db-us-east-1a"
          + "Region"      = "east"          
        }
      + vpc_id                          = (known after apply)
    }

  # module.vpc.aws_subnet.database[1] will be created
  + resource "aws_subnet" "database" {
      + arn                             = (known after apply)
      + assign_ipv6_address_on_creation = false
      + availability_zone               = "us-east-1b"
      + availability_zone_id            = (known after apply)
      + cidr_block                      = "10.0.9.0/24"
      + id                              = (known after apply)
      + ipv6_cidr_block                 = (known after apply)
      + ipv6_cidr_block_association_id  = (known after apply)
      + map_public_ip_on_launch         = false
      + owner_id                        = (known after apply)
      + tags                            = {
          + "Environment" = "dev"
          + "Name"        = "dev-vpc-db-us-east-1b"
          + "Region"      = "east"          
        }
      + vpc_id                          = (known after apply)
    }

  # module.vpc.aws_subnet.public[0] will be created
  + resource "aws_subnet" "public" {
      + arn                             = (known after apply)
      + assign_ipv6_address_on_creation = false
      + availability_zone               = "us-east-1a"
      + availability_zone_id            = (known after apply)
      + cidr_block                      = "10.0.0.0/24"
      + id                              = (known after apply)
      + ipv6_cidr_block                 = (known after apply)
      + ipv6_cidr_block_association_id  = (known after apply)
      + map_public_ip_on_launch         = true
      + owner_id                        = (known after apply)
      + tags                            = {
          + "Environment" = "dev"
          + "Name"        = "dev-vpc-public-us-east-1a"
          + "Region"      = "east"          
        }
      + vpc_id                          = (known after apply)
    }

  # module.vpc.aws_subnet.public[1] will be created
  + resource "aws_subnet" "public" {
      + arn                             = (known after apply)
      + assign_ipv6_address_on_creation = false
      + availability_zone               = "us-east-1b"
      + availability_zone_id            = (known after apply)
      + cidr_block                      = "10.0.1.0/24"
      + id                              = (known after apply)
      + ipv6_cidr_block                 = (known after apply)
      + ipv6_cidr_block_association_id  = (known after apply)
      + map_public_ip_on_launch         = true
      + owner_id                        = (known after apply)
      + tags                            = {
          + "Environment" = "dev"
          + "Name"        = "dev-vpc-public-us-east-1b"
          + "Region"      = "east"          
        }
      + vpc_id                          = (known after apply)
    }

  # module.vpc.aws_vpc.this[0] will be created
  + resource "aws_vpc" "this" {
      + arn                              = (known after apply)
      + assign_generated_ipv6_cidr_block = false
      + cidr_block                       = "10.0.0.0/16"
      + default_network_acl_id           = (known after apply)
      + default_route_table_id           = (known after apply)
      + default_security_group_id        = (known after apply)
      + dhcp_options_id                  = (known after apply)
      + enable_classiclink               = (known after apply)
      + enable_classiclink_dns_support   = (known after apply)
      + enable_dns_hostnames             = false
      + enable_dns_support               = true
      + id                               = (known after apply)
      + instance_tenancy                 = "default"
      + ipv6_association_id              = (known after apply)
      + ipv6_cidr_block                  = (known after apply)
      + main_route_table_id              = (known after apply)
      + owner_id                         = (known after apply)
      + tags                             = {
          + "Environment" = "dev"
          + "Name"        = "dev-vpc"
          + "Region"      = "east"          
        }
    }

Plan: 15 to add, 0 to change, 0 to destroy.

------------------------------------------------------------------------

This plan was saved to: dev-state.tf

To perform exactly these actions, run the following command to apply:
    terraform apply "dev-state.tf"
```

Here I am applying the above state and spinning up the infrastructure.

```bash
OpsToDevOps : terraform apply "dev-state.tf"
data.aws_availability_zones.azs: Refreshing state...
module.vpc.aws_vpc.this[0]: Creating...
module.vpc.aws_vpc.this[0]: Creation complete after 1s [id=vpc-0c04ce9bd36766ea0]
module.vpc.aws_route_table.private[1]: Creating...
module.vpc.aws_subnet.database[0]: Creating...
module.vpc.aws_subnet.database[1]: Creating...
module.vpc.aws_subnet.public[0]: Creating...
module.vpc.aws_route_table.private[0]: Creating...
module.vpc.aws_subnet.public[1]: Creating...
module.vpc.aws_route_table.public[0]: Creating...
module.vpc.aws_internet_gateway.this[0]: Creating...
module.vpc.aws_route_table.public[0]: Creation complete after 0s [id=rtb-0d7bca8ba09b4e666]
module.vpc.aws_route_table.private[1]: Creation complete after 0s [id=rtb-05f281bc170956365]
module.vpc.aws_route_table.private[0]: Creation complete after 0s [id=rtb-01e5a96614a8080f7]
module.vpc.aws_internet_gateway.this[0]: Creation complete after 0s [id=igw-0fbbdf73904d095db]
module.vpc.aws_route.public_internet_gateway[0]: Creating...
module.vpc.aws_route.public_internet_gateway[0]: Creation complete after 1s [id=r-rtb-0d7bca8ba09b4e6661080289494]
module.vpc.aws_subnet.database[0]: Creation complete after 1s [id=subnet-0bd091b9bde547cd4]
module.vpc.aws_subnet.database[1]: Creation complete after 1s [id=subnet-097096229c9e56d81]
module.vpc.aws_route_table_association.database[0]: Creating...
module.vpc.aws_route_table_association.database[1]: Creating...
module.vpc.aws_db_subnet_group.database[0]: Creating...
module.vpc.aws_subnet.public[1]: Creation complete after 1s [id=subnet-056b3e3be91c96eff]
module.vpc.aws_subnet.public[0]: Creation complete after 1s [id=subnet-0d13251839f0922df]
module.vpc.aws_route_table_association.public[1]: Creating...
module.vpc.aws_route_table_association.public[0]: Creating...
module.vpc.aws_route_table_association.database[1]: Creation complete after 0s [id=rtbassoc-0da465c5b0a9f821c]
module.vpc.aws_route_table_association.database[0]: Creation complete after 0s [id=rtbassoc-02d423612cd6fc1af]
module.vpc.aws_route_table_association.public[0]: Creation complete after 0s [id=rtbassoc-0897e3249000cdb0f]
module.vpc.aws_route_table_association.public[1]: Creation complete after 0s [id=rtbassoc-051a44de6e1e1dad8]
module.vpc.aws_db_subnet_group.database[0]: Creation complete after 1s [id=dev-vpc]

Apply complete! Resources: 15 added, 0 changed, 0 destroyed.

Outputs:

db_subnet_group = module.vpc.database_subnet_group
public_subnets = module.vpc.public_subnets
vpc_id = module.vpc.vpc.id
```
Let’s check if the S3 Bucket created earlier has some content in it. I can see by listing the contents of the bucket that indeed there is a `terraform.tfstate` file.

```bash
OpsToDevOps : aws s3 ls ops2dev-tfstatebucket-33444 --recursive
2020-07-07 21:06:51      50001 terraform-state/dev-vpc/terraform.tfstate
```

And while we are at it, let me check the DynamoDB table as well.

```bash
OpsToDevOps : aws dynamodb list-tables --output table
---------------------------------
|          ListTables           |
+-------------------------------+
||         TableNames          ||
|+-----------------------------+|
||  ops2dev-tfstatelock-33444  ||
|+-----------------------------+|
OpsToDevOps :
```
That seems all in place, so what happens when another user attempts to make changes to the infrastructure while I was planning and applying my changes?\
Jane Doe here is running a plan while I was building my infrastructure & as is evident that Jane will have to wait till my actions are completed and saved. 
That is because we are using DynamoDB as a locking mechanism and DynamoDB has put a state lock on the file till another operation is in progress.

```bash
Jane Doe : terraform plan

Error: Error locking state: Error acquiring the state lock: ConditionalCheckFailedException: The conditional request failed
Lock Info:
  ID:        8f588a82-e2f5-25ef-368c-158f365af586
  Path:      ops2dev-tfstatebucket-33444/terraform-state/dev-vpc/terraform.tfstate
  Operation: OperationTypeApply
  Who:       ubuntu@ip-172-31-74-205
  Version:   0.12.28
  Created:   2020-07-07 21:25:09.059492135 +0000 UTC
  Info:      


Terraform acquires a state lock to protect the state from being written
by multiple users at the same time. Please resolve the issue above and try
again. For most commands, you can disable locking with the "-lock=false"
flag, but this is not recommended.


Jane Doe :
```
So that is it for now and if you have been following along the demo in this blog, make sure to clean up to avoid incurring charges.

```bash
OpsToDevOps : terraform destroy --auto-approve
```

You can read more about [remote state](https://www.terraform.io/docs/state/remote.html), using [S3 & DynamoDB as backend](https://www.terraform.io/docs/backends/types/s3.html) and [state locking](https://www.terraform.io/docs/state/locking.html)

Hope you enjoyed this post, stay safe!


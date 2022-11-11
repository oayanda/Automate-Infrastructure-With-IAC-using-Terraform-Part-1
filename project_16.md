# CONFIGURE AWS CLI

***`Prerequisite`***

A AWS IAM User with a Programmatic access. (The User should have a access key id and access key)

***`Install AWS CLI for windows`***

Open the command terminal with administrator's priviledge and run the command below

```bash
msiexec.exe /i https://awscli.amazonaws.com/AWSCLIV2.msi

#verify installtion
aws --version
```

[*`click here for instructions to install AWS CLI on other OS`*]( https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html )

Configure AWS CLI in the terminal

```bash
> aws configure
```

Follow the prompt, by inputing your access key id, access key and region and press enter.

Install Boto3 (Boto3 is a AWS SDK for Python)

```bash
pip install boto3
```

Terraform must store state about your managed infrastructure and configuration. This state is used by Terraform to map real world resources to your configuration, keep track of metadata, and to improve performance for large infrastructures.

This state is stored by default in a local file named "terraform.tfstate", but it can also be stored remotely, which works better in a team environment.

Create a S3 bucket resource to store terraform state remotely. 
Type `S3` in search bar in aws management console
Click on create bucket
![Amazon S3 bucket](/images/1.png)

Enter a Bucket name, AWS Region, Enable Bucket Versioning, Add a Tag and click create bucket.
![Amazon S3 bucket](/images/2.png)

You can also verfiy this in the AWS CLI

```bash
aws s3 ls
```

![Amazon S3 bucket](/images/3.png)

## Configuring AWS VPC and Subnets as Code

Install Terraform on Windows

```bash
choco install terraform

# Verify installation
terraform -help
```

Let us create a directory structure
Open your Visual Studio Code and:

Create a folder called PBL
Create a file in the folder, name it main.tf

![Amazon S3 bucket](/images/4.png)

Add `AWS` as a `provider` code block, and a `resource` code block to create a VPC in the `main.tf` file.

> `Provider` block informs Terraform that we intend to build infrastructure within AWS and
Resource block will create a VPC.

```bash
provider "aws" {
  region = "us-west-2"
}

# Create VPC
resource "aws_vpc" "main" {
  cidr_block                     = "172.16.0.0/16"
  enable_dns_support             = "true"
  enable_dns_hostnames           = "true"
}
```

> With the retirement of EC2-Classic the enable_classiclink attribute has been deprecated and will be removed in a future version.

The next thing we need to do, is to download necessary plugins for Terraform to work. These plugins are used by providers and provisioners. At this stage, we only have `provider` in our `main.tf`file. So, Terraform will just download plugin for AWS provider.
Navigate to the PBL folder 

```bash
> cd PBL

> terraform init
```

![Amazon S3 bucket](/images/5.png)

Let's verify what terraform intends to create

```bash
terraform plan

# If it looks okay go ahead and create the resource

terraform apply
```

![Amazon S3 bucket](/images/6.png)

## Subnets resource section

According to our architectural design, 6 subnets are required:

2 public subnets
2 private subnets for webservers
2 private subnets for data layer
Let us create the first 2 public subnets.

Add below configuration to the main.tf file:

```bash

# Create public subnets1
    resource "aws_subnet" "public1" {
    vpc_id                     = aws_vpc.main.id
    cidr_block                 = "172.16.0.0/24"
    map_public_ip_on_launch    = true
    availability_zone          = "us-west-2a"

}

# Create public subnet2
    resource "aws_subnet" "public2" {
    vpc_id                     = aws_vpc.main.id
    cidr_block                 = "172.16.1.0/24"
    map_public_ip_on_launch    = true
    availability_zone          = "us-west-2b"
}
```

Run `terraform plan` to check the intend infrusture and `terraform apply` to create the infrusture.
![Amazon S3 bucket](/images/7.png)

Note - `Plan: 3 to add, 0 to change, 0 to destroy.` This refers to the vpc and the 2 subnets to be created. Type Yes to aspect the creation.
Verify Vpc in aws management console
![Amazon S3 bucket](/images/vpc.png)

Verify subnets in aws management console
![Amazon S3 bucket](/images/sub.png)

However, this process is not dynamic and can become tideous when working with huge infrustures. We need to optimize this by introducing a count argument and refactoring our code.

First, destroy the current infrastructure. Since we are still in development, this is totally fine. Otherwise, `DO NOT DESTROY` an infrastructure that has been deployed to production.

```bash
terraform destroy
```

## Fixing The Problems By Code Refactoring

As stated ealier, while the code above worked fine, the process is inefficient and not dynamic. Some o :f the problems we will be solving includes the following

- Fixing Hard Coded Values
- Fixing multiple resource blocks
- Let’s make cidr_block dynamic

*`Fixing the Hard Coded Values`*

We will introduce variables, and remove hard coding.

Starting with the `provider` block, declare a `variable` named `region`, give it a default value (if you don't declare a default value, you will be prompted each time you run terraform plan/apply), and update the provider section by referring to the declared variable.

```bash
 variable "region" {
        default = "us-west-2"
    }

    provider "aws" {
        region = var.region
    }
```

Do the same to cidr value in the vpc block, and all the other arguments.

```bash
  variable "region" {
        default = "us-west-2"
    }

    variable "vpc_cidr" {
        default = "172.16.0.0/16"
    }

    variable "enable_dns_support" {
        default = "true"
    }

    variable "enable_dns_hostnames" {
        default ="true" 
    }


    provider "aws" {
    region = var.region
    }

    # Create VPC
    resource "aws_vpc" "main" {
    cidr_block                     = var.vpc_cidr
    enable_dns_support             = var.enable_dns_support 
    enable_dns_hostnames           = var.enable_dns_support
   
    }
```

*`Fixing multiple resource blocks`*

Terraform has a functionality that allows us to pull data which exposes information to us. Terraform’s Data Sources helps us to fetch information outside of Terraform. In this case, from AWS.

Let us fetch Availability zones from AWS, and replace the hard coded value in the subnet’s availability_zone section.

```bash
  # Get list of availability zones
        data "aws_availability_zones" "available" {
        state = "available"
        }
```

To make use of this new data resource, we will need to introduce a count argument in the subnet block: Something like this.

```bash
    # Create public subnet
    resource "aws_subnet" "public" { 
        count                   = 2
        vpc_id                  = aws_vpc.main.id
        cidr_block              = "172.16.1.0/24"
        map_public_ip_on_launch = true
        availability_zone       = data.aws_availability_zones.available.names[count.index]

    }
```

However, the cidr_block needs to be dynamic and this can be acheieved by introducing the function cidrsubnet() which accepts three parameter - cidrsubnet(prefix, newbits, netnum)

- The `prefix` parameter must be given in CIDR notation, same as for VPC.
- The `newbits` parameter is the number of additional bits with which to extend the prefix. For example, if given a prefix ending with /16 and a newbits value of 4, the resulting subnet address will have length /20
- The -  parameter is a whole number that can be represented as a binary integer with no more than newbits binary digits, which will be used to populate the additional bits added to the prefix

The snippent below would now replace the 2 resource blocks we created earlier. As well this single code snippet can create mulitiply subnets by just replacing the `count` value accordingly.

```bash
    # Create public subnet
    resource "aws_subnet" "public" { 
        count                   = 2
        vpc_id                  = aws_vpc.main.id
        cidr_block              = cidrsubnet(var.vpc_cidr, 4 , count.index)
        map_public_ip_on_launch = true
        availability_zone       = data.aws_availability_zones.available.names[count.index]

    }
```

*`Removing hard coded count value.`*

Additionally, instead of hard coding the count value, we can use the length(), create variable and set a default value for count.

```bash
# Create public subnets
resource "aws_subnet" "public" {
  count  = var.preferred_number_of_public_subnets == null ? length(data.aws_availability_zones.available.names) : var.preferred_number_of_public_subnets   
  vpc_id = aws_vpc.main.id
  cidr_block              = cidrsubnet(var.vpc_cidr, 4 , count.index)
  map_public_ip_on_launch = true
  availability_zone       = data.aws_availability_zones.available.names[count.index]

}
```

```bash
# new variable and default value for count
variable "preferred_number_of_public_subnets" {
  default = 2
}

```

Now lets break it down:

- The first part `var.preferred_number_of_public_subnets == null` checks if the value of the variable is set to null or has some value defined.
- The second part `?` and `length(data.aws_availability_zones.available.names)` means, if the first part is true, then use this. In other words, if preferred number of public subnets is null (Or not known) then set the value to the data returned by lenght function.
- The third part : and `var.preferred_number_of_public_subnets` means, if the first condition is false, i.e preferred number of public subnets is not null then set the value to whatever is definied in `var.preferred_number_of_public_subnets`


Now the entire configuration should now look like this

```bash
# Get list of availability zones
data "aws_availability_zones" "available" {
state = "available"
}

variable "region" {
      default = "eu-central-1"
}

variable "vpc_cidr" {
    default = "172.16.0.0/16"
}

variable "enable_dns_support" {
    default = "true"
}

variable "enable_dns_hostnames" {
    default ="true" 
}

variable "enable_classiclink" {
    default = "false"
}

variable "enable_classiclink_dns_support" {
    default = "false"
}

  variable "preferred_number_of_public_subnets" {
      default = 2
}

provider "aws" {
  region = var.region
}

# Create VPC
resource "aws_vpc" "main" {
  cidr_block                     = var.vpc_cidr
  enable_dns_support             = var.enable_dns_support 
  enable_dns_hostnames           = var.enable_dns_support
  enable_classiclink             = var.enable_classiclink
  enable_classiclink_dns_support = var.enable_classiclink

}

# Create public subnets
resource "aws_subnet" "public" {
  count  = var.preferred_number_of_public_subnets == null ? length(data.aws_availability_zones.available.names) : var.preferred_number_of_public_subnets   
  vpc_id = aws_vpc.main.id
  cidr_block              = cidrsubnet(var.vpc_cidr, 4 , count.index)
  map_public_ip_on_launch = true
  availability_zone       = data.aws_availability_zones.available.names[count.index]

}
```
![Amazon S3 bucket](/images/8.png)

Now instead of havng a long file in main.tf file, we can actually make our code a lot more readable and better structured by moving out some parts of the configuration content to other files.

Create two files , 
variables.tf - to hold all variables
terraform.tfvars - to hold the default values for the variables.

***`main.tf`***
![Amazon S3 bucket](/images/main.png)

***`variables.tf`***
![Amazon S3 bucket](/images/variables.png)

***`terraform.tfvars`***
![Amazon S3 bucket](/images/terraform.png)


Create the Network infrastructure

```bash

# verify the intended infrastructure to be created
terraform plan
```

![Amazon S3 bucket](/images/plan.png)

```bash
# To create the infrastructure
terraform apply
```
![Amazon S3 bucket](/images/apply.png)
![Amazon S3 bucket](/images/apply2.png)

Verify VPC in AWS management Console
![Amazon S3 bucket](/images/vpc2.png)

Verify Subnet in AWS management Console
![Amazon S3 bucket](/images/sub2.png)

```bash
# Delete the infastructure
terraform destory
```
![Amazon S3 bucket](/images/dest.png)

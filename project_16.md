# CONFIGURE AWS CLI

***`Prerequisite`***

A AWS IAM User with a Programmatic access. (The User has a access key id and access key)

***`Install AWS CLI for windows`***

Open the command terminal with administrator's priviledge and run the command below

```bash
msiexec.exe /i https://awscli.amazonaws.com/AWSCLIV2.msi

#verify installtion
aws --version
```

[*`click here for instructions to install AWS CLI on other OS`*]( https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html )

Configure AWS CLI

```bash
aws configure
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


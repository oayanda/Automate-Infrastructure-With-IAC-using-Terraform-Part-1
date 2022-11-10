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
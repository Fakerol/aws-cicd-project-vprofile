# AWS CI/CD Project Setup Guide

This guide outlines the steps to set up a CI/CD pipeline using AWS services, BitBucket, and a sample application. The project includes AWS Elastic Beanstalk, Amazon RDS, AWS CodeBuild, and AWS CodePipeline, with code migration from GitHub to BitBucket.

## Table of Contents
1. [Prerequisites](#prerequisites)
2. [AWS Elastic Beanstalk Setup](#aws-elastic-beanstalk-setup)
3. [Amazon RDS Setup](#amazon-rds-setup)
4. [Database Initialization and EC2-RDS Connection](#database-initialization-and-ec2-rds-connection)
5. [BitBucket Repository Setup and Migration](#bitbucket-repository-setup-and-migration)
6. [AWS CodeBuild Configuration](#aws-codebuild-configuration)
7. [AWS CodePipeline Setup](#aws-codepipeline-setup)
8. [Testing the CI/CD Flow](#testing-the-cicd-flow)

## Prerequisites
- AWS account with access to Elastic Beanstalk, RDS, CodeBuild, CodePipeline, and S3.
- BitBucket account.
- Git installed locally.
- SSH key pair for BitBucket.
- EC2 key pair in `.pem` format.

## AWS Elastic Beanstalk Setup
1. Log in to the AWS Management Console.
2. Navigate to **Elastic Beanstalk** > **Create Application**.
3. Follow the setup instructions as shown in the `beanstalk-setup` screenshots (refer to project folder).
4. Deploy the application environment.

## Amazon RDS Setup
1. Navigate to **Amazon RDS** > **Create Database**.
2. Configure the database (e.g., MySQL) as per the `rds` screenshots (refer to project folder).
3. Note the RDS endpoint (e.g., `my-rds.example-region.rds.amazonaws.com`).

## Database Initialization and EC2-RDS Connection
### Allow RDS to Receive Traffic from EC2
1. Go to **EC2** > **Security Groups** and copy the EC2 security group ID (e.g., `sg-1234567890abcdef0`).
2. Navigate to **RDS** > **Security Groups** > Edit inbound rules.
3. Add a rule: **Custom TCP**, Port **3306**, Source **Custom**, paste the EC2 security group ID.
4. Save the changes.

### Test Connection to RDS
1. Copy the **Public IPv4 address** of an EC2 instance from the AWS Console (e.g., `192.168.1.100`).
2. Open a terminal (e.g., Git Bash) and connect to the EC2 instance:
   ```bash
   ssh -i my-key.pem ec2-user@192.168.1.100
   ```
3. Install MySQL client:
   ```bash
   dnf search mysql
   sudo dnf install mariadb105 -y
   ```
4. Test the RDS connection:
   ```bash
   mysql -h my-rds.example-region.rds.amazonaws.com -u dbadmin -p
   ```
5. Enter the database password (e.g., `mypassword123`).
6. Select the database:
   ```sql
   USE accounts;
   ```

### Import Database
1. Download the sample database:
   ```bash
   wget https://raw.githubusercontent.com/sample-user/sample-project/main/src/main/resources/db_backup.sql
   ```
2. Import the database:
   ```bash
   mysql -h my-rds.example-region.rds.amazonaws.com -u dbadmin -p accounts < db_backup.sql
   ```
3. Verify the import:
   ```bash
   mysql -h my-rds.example-region.rds.amazonaws.com -u dbadmin -p accounts
   show tables;
   ```

## BitBucket Repository Setup and Migration
### BitBucket Setup
1. Register at [bitbucket.org](https://bitbucket.org).
2. Create a workspace and a new repository (e.g., `my-app`).

### Add SSH Key to BitBucket
1. Generate an SSH key locally:
   ```bash
   ssh-keygen
   ```
   - Save the key as `my-bit-rsa`.
   - Leave the passphrase empty.
2. Display the public key:
   ```bash
   cat ~/.ssh/my-bit-rsa.pub
   ```
   Example output: `ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAI...`
3. Add the public key to BitBucket:
   - Go to **Settings** > **SSH Keys** > **Add Key** at [bitbucket.org/account/settings/ssh-keys/](https://bitbucket.org/account/settings/ssh-keys/).
4. Update the SSH configuration:
   ```bash
   vim ~/.ssh/config
   ```
   Add:
   ```
   Host bitbucket.org
       HostName bitbucket.org
       User git
       IdentityFile ~/.ssh/my-bit-rsa
   ```

### Migrate from GitHub to BitBucket
1. Clone the GitHub repository:
   ```bash
   git clone https://github.com/sample-user/sample-project.git
   cd sample-project
   git checkout aws-ci
   git fetch --tags
   ```
2. Remove the GitHub remote:
   ```bash
   git remote rm origin
   ```
3. Add the BitBucket remote:
   ```bash
   git remote add origin git@bitbucket.org:my-workspace/my-app.git
   ```
4. Push to BitBucket:
   ```bash
   git push origin --all
   ```

## AWS CodeBuild Configuration
1. Create an S3 bucket for artifacts (e.g., `my-cicd-artifact-bucket`).
2. Navigate to **AWS CodeBuild** > **Create Build Project**.
3. Configure the source:
   - Select **BitBucket** as the source provider.
   - Use the default credentials for BitBucket.
4. Add the following `buildspec.yml`:
```bash
version: 0.2

phases:
  install:
    runtime-versions:
      java: corretto17
  pre_build:
    commands:
      - apt-get update
      - apt-get install -y jq
      - wget https://archive.apache.org/dist/maven/maven-3/3.9.8/binaries/apache-maven-3.9.8-bin.tar.gz
      - tar xzf apache-maven-3.9.8-bin.tar.gz
      - ln -s apache-maven-3.9.8 maven
      - sed -i 's/jdbc.password=admin123/jdbc.password=mypassword123/' src/main/resources/application.properties
      - sed -i 's/jdbc.username=admin/jdbc.username=dbadmin/' src/main/resources/application.properties
      - sed -i 's/db01:3306/my-rds.example-region.rds.amazonaws.com:3306/' src/main/resources/application.properties
  build:
    commands:
      - mvn install
  post_build:
    commands:
      - mvn package
artifacts:
  files:
    - '**/*'
  base-directory: 'target/vprofile-v2'
```

## AWS CodePipeline Setup

1. Navigate to AWS CodePipeline > Create Pipeline.
2. Select Build Custom Pipeline.
3. Configure the pipeline:
   - Source: BitBucket repository (my-app).
   - Build: Select the CodeBuild project created above.
   - Deploy: Select Elastic Beanstalk and specify the application/environment.

4. Create and run the pipeline.

5. Testing the CI/CD Flow

6. Make a change to the codebase in the BitBucket repository.
   - Commit and push the changes:
```bash
git add .
git commit -m "Test CI/CD pipeline"
git push origin aws-ci
```

7. Monitor the pipeline in AWS CodePipeline to ensure the build and deployment complete successfully.
8. ;Verify the application is updated in the Elastic Beanstalk environment.

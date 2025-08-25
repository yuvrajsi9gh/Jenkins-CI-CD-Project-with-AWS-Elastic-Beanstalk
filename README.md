# Jenkins CI/CD Project with AWS Elastic Beanstalk and GitHub

## Overview
In this project, we create a **Jenkins CI/CD pipeline** to deploy an application to **AWS Elastic Beanstalk** whenever changes are pushed to a **GitHub repository**.  
We will set up and demonstrate this in two ways:
1. **Freestyle Project (UI-driven)**
2. **Pipeline Project (Pipeline-as-Code using Jenkinsfile)**

---

## 1. Prerequisites

### Resources Required
- **Jenkins Server** (Amazon EC2, t2.medium, 20GB Storage)
- **AWS Elastic Beanstalk Application**
- **GitHub Repository**

---

## 2. Jenkins Server Setup

### Step 1: Launch and Configure Jenkins
1. Launch an EC2 instance:
   - **Type**: t2.medium  
   - **Storage**: 20GB  
   - **OS**: Amazon Linux 2 / RHEL  

2. Install Jenkins:
   ```bash
   yum install wget -y
   wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat-stable/jenkins.repo
   rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io-2023.key
   yum upgrade -y
   yum install -y fontconfig java-21-openjdk
   yum install -y jenkins
   systemctl daemon-reload
   systemctl enable --now jenkins
   ````

3. Get the initial Jenkins admin password:

   ```bash
   cat /var/lib/jenkins/secrets/initialAdminPassword
   ```

### Step 2: Basic Tools

Install Git and Zip utilities:

```bash
yum install -y git zip
````

### Step 3: Access Jenkins Web UI

* Open: `http://<EC2-Public-IP>:8080`
* Paste the initial admin password, create an admin user, and install suggested plugins.

---

## 3. Jenkins Configuration

### Step 1: Install Required Plugins

* **AWSEB Deployment Plugin**
* **NodeJS Plugin**
* **Pipeline: Stage View Plugin**

---

### Step 2: Add AWS Credentials

We need to provide Jenkins with AWS credentials so it can deploy to Elastic Beanstalk.

#### **2.1 Create an IAM User in AWS**

1. Login to **AWS Management Console** → Go to **IAM**.
2. Create a New User:

   * **Users → Add Users**
   * Enter username (e.g., `jenkins-eb-user`)
   * Select **Access key – Programmatic access**
3. Assign Permissions:

   * Attach existing policies directly:

     * `AdministratorAccess-AWSElasticBeanstalk`
     * `AmazonS3FullAccess`
4. Create User and **Download Access Key & Secret Key**.

> **Tip:** Keep these credentials safe; we will use them in Jenkins.


#### **2.2 Add AWS Credentials in Jenkins**

1. Go to **Manage Jenkins → Credentials → Global → Add Credentials**
2. **Kind**: AWS Credentials
3. Enter:

   * **ID**: `aws-jenks`
   * **Access Key ID** & **Secret Access Key**
4. Click **OK**

---

### Step 3: Configure NodeJS Tool

* **Manage Jenkins > Global Tool Configuration > NodeJS**

  * **Name**: `nodejs`
  * **Version**: NodeJS 22.8.0

---

## 4. AWS Setup

1. Create an **Elastic Beanstalk Application**:

   * Platform: Node.js (or Java if using Java app)
   * Environment: Web Server environment

2. Note the **Application Name** and **Environment Name** (used later in Jenkins).

---

## 5. GitHub Setup

1. Create/Use an existing repository with your application code.
2. Enable **Webhooks**:

   * Go to **Settings > Webhooks > Add Webhook**
   * **URL**: `http://<Jenkins-Server-IP>:8080/github-webhook/`
   * **Event**: Push events

---

# Part A: Freestyle Project Setup

### Step 1: Create a Freestyle Job

* **New Item > Freestyle Project > Name**: `Freestyle-EB-Deploy`

### Step 2: Configure Source Code Management

* Select **Git**
* URL: `https://github.com/<your-repo>.git`
* Branch: `main`

### Step 3: Build Environment

* Add **Execute Shell**:

  ```bash
  npm install
  npm run build
  zip -r app.zip * .[^.]*
  ```

### Step 4: Build Action: Deploy to Elastic Beanstalk

* Add Build Action: **Deploy Application to AWS Elastic Beanstalk**

  * **AWS Credentials ID**: `aws-jenks`
  * **AWS Region**: `ap-south-1`
  * **Application Name**: `<Your EB App>`
  * **Environment Name**: `<Your EB Env>`
  * **S3 Key Prefix**: `<app-name>/builds`
  * **Root Object**: `app.zip`

### Step 5: Trigger Builds

* Enable **GitHub hook trigger for GITScm polling**

### Step 6: Test

* Push code to GitHub → Jenkins pulls, builds, and deploys to EB.

---

# Part B: Pipeline Project (Pipeline-as-Code)

### Step 1: Create a Pipeline Job

* **New Item > Pipeline > Name**: `Pipeline-EB-Deploy`

### Step 2: Pipeline Script

Use this **Jenkinsfile** (Declarative Pipeline):

```groovy
pipeline {
    agent any

    triggers {
        githubPush()
    }

    environment {
        git_origin = 'https://github.com/<your-repo>.git'
        aws_region = 'ap-south-1'
        aws_credentials = 'aws-jenks'
        aws_application_name = 'Pipeline-project'
        aws_environment_name = 'Pipeline-project-env'
        aws_root_object = 'app.zip'
        aws_key_prefix = 'Pipeline-project/builds'
    }

    tools { nodejs 'nodejs' }

    stages {
        stage('Source Code from GitHub to Jenkins') {
            steps {
                sh 'env'
                git branch: 'main', url: git_origin
            }
        }

        stage('Building Package from our Source Code') {
            steps {
                sh '''
                    npm install
                    npm run build
                    zip -r app.zip * .[^.]*
                '''
            }
        }

        stage('Application deploying to Elastic Beanstalk') {
            steps {
                step([$class: 'AWSEBDeploymentBuilder',
                    credentialId: aws_credentials,
                    awsRegion: aws_region,
                    applicationName: aws_application_name,
                    environmentName: aws_environment_name,
                    keyPrefix: aws_key_prefix,
                    rootObject: aws_root_object,
                    versionLabelFormat: "-${BUILD_NUMBER}"
                ])
            }
        }
    }
}
```

### Step 3: Trigger

* Pipeline triggers on every **push to GitHub**.

### Step 4: Test

* Push code → Pipeline runs → EB updates automatically.

---

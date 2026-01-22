# AWS CodePipeline Deployment Guide for eb-tomcat-snakes (Ubuntu EC2)

This guide will walk you through deploying your Java web application to an **Ubuntu EC2** server using AWS CodePipeline.

## Prerequisites (Ubuntu EC2)

Before starting the pipeline setup, your EC2 instance must be ready.

1.  **Launch an EC2 Instance** (Ubuntu 20.04 or 22.04 LTS).
2.  **Update and Install Java**:
    ```bash
    sudo apt update
    sudo apt install openjdk-11-jdk -y
    java -version
    ```
3.  **Install Tomcat (Manual Install)**:
    We install manually to `/opt/tomcat` so it matches the configuration in `appspec.yml`.
    ```bash
    cd /opt
    # Download Tomcat 9
    sudo wget https://archive.apache.org/dist/tomcat/tomcat-9/v9.0.98/bin/apache-tomcat-9.0.98.tar.gz
    # Extract
    sudo tar -xvf apache-tomcat-9.0.98.tar.gz
    sudo mv apache-tomcat-9.0.98 tomcat
    sudo rm apache-tomcat-9.0.98.tar.gz
    # Set permissions
    sudo chmod +x /opt/tomcat/bin/*.sh
    ```
4.  **Create a Tomcat Service**:
    You need a systemd service so the deployment scripts can run `systemctl start/stop tomcat`.
    *   Create file: `sudo nano /etc/systemd/system/tomcat.service`
    *   Paste the following:
        ```ini
        [Unit]
        Description=Apache Tomcat Web Application Container
        After=network.target

        [Service]
        Type=forking
        Environment=JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64
        Environment=CATALINA_PID=/opt/tomcat/temp/tomcat.pid
        Environment=CATALINA_HOME=/opt/tomcat
        Environment=CATALINA_BASE=/opt/tomcat
        ExecStart=/opt/tomcat/bin/startup.sh
        ExecStop=/opt/tomcat/bin/shutdown.sh
        User=root
        Group=root

        [Install]
        WantedBy=multi-user.target
        ```
    *   **Note**: Verify your `JAVA_HOME` with `update-java-alternatives -l` if needed.
    *   Enable and start:
        ```bash
        sudo systemctl daemon-reload
        sudo systemctl start tomcat
        sudo systemctl enable tomcat
        ```
5.  **Install CodeDeploy Agent (Ubuntu)**:
    ```bash
    sudo apt install ruby-full wget -y
    cd /home/ubuntu
    # Replace <YOUR_REGION> with your actual region, e.g., us-east-1
    wget https://aws-codedeploy-<YOUR_REGION>.s3.<YOUR_REGION>.amazonaws.com/latest/install
    chmod +x ./install
    sudo ./install auto
    sudo service codedeploy-agent status
    ```
6.  **IAM Role for EC2**:
    *   Create an IAM Role (e.g., `EC2CodeDeployRole`).
    *   Attach policies: `AmazonEC2RoleforAWSCodeDeploy` and `AmazonS3ReadOnlyAccess`.
    *   Attach this role to your EC2 instance.

---

## Step 1: Create CodeDeploy Application

1.  Go to **AWS CodeDeploy** console -> **Applications** -> **Create application**.
2.  **Application name**: `SnakesApp`.
3.  **Compute Platform**: `EC2/On-premises`.
4.  Click **Create application**.
5.  Inside the app, click **Create deployment group**.
    *   **Name**: `SnakesDepGroup`.
    *   **Service Role**: Choose a role with `AWSCodeDeployRole` policy.
    *   **Environment configuration**: Select **Amazon EC2 instances** and select your instance Key/Value tag.
    *   **Load Balancer**: Uncheck "Enable load balancing" (for simplicity).
    *   Click **Create deployment group**.

---

## Step 2: Create a Build Project (CodeBuild)

1.  Go to **AWS CodeBuild** -> **Create build project**.
2.  **Project name**: `SnakesBuild`.
3.  **Source**: Your CodeCommit/GitHub repo.
4.  **Environment**:
    *   **Managed image**.
    *   **OS**: `Amazon Linux 2` (CodeBuild runs in a container, so AL2 is fine/standard for the build itself).
    *   **Runtime**: `Standard`.
    *   **Image**: `aws/codebuild/amazonlinux2-x86_64-standard:4.0`.
5.  **Buildspec**: "Use a buildspec file".
6.  Click **Create build project**.

---

## Step 3: Create the Pipeline

1.  Go to **AWS CodePipeline** -> **Create pipeline**.
2.  **Pipeline name**: `SnakesPipeline`.
3.  **Source Stage**: Your Repo.
4.  **Build Stage**: `SnakesBuild`.
5.  **Deploy Stage**: `AWS CodeDeploy` -> `SnakesApp` -> `SnakesDepGroup`.
6.  **Create pipeline**.

---

## Verification

After deploy:
Open `http://<EC2-Public-IP>:8080/`

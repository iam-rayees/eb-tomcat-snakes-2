# AWS CodePipeline Deployment Guide for eb-tomcat-snakes (Ubuntu EC2)

This guide will walk you through deploying your Java web application to an **Ubuntu EC2** server using AWS CodePipeline.

## Prerequisites (Ubuntu EC2)

Before starting the pipeline setup, your EC2 instance must be ready.

1.  **Launch an EC2 Instance** (Ubuntu 20.04 or 22.04 LTS).
2.  **Install Java and Tomcat 10 (User Provided Script)**:
    Run the following commands on your EC2 instance:
    ```bash
    sudo apt update
    sudo apt install default-jdk -y
    sudo apt upgrade -y
    java --version
    
    # Create tomcat user
    sudo useradd -m -d /opt/tomcat -U -s /bin/false tomcat
    
    # Install Tomcat 10
    cd /tmp
    sudo wget https://archive.apache.org/dist/tomcat/tomcat-10/v10.1.24/bin/apache-tomcat-10.1.24.tar.gz -O /tmp/tomcat-10.tar.gz
    sudo -u tomcat tar -xzvf /tmp/tomcat-10.tar.gz --strip-components=1 -C /opt/tomcat
    ```
3.  **Create Tomcat Service**:
    *   Create the file: `sudo nano /etc/systemd/system/tomcat.service`
    *   Paste the exact content below:
        ```ini
        [Unit]
        Description=Apache Tomcat
        After=network.target

        [Service]
        Type=forking

        User=tomcat
        Group=tomcat

        Environment=JAVA_HOME=/usr/lib/jvm/java-1.21.0-openjdk-amd64
        Environment=CATALINA_PID=/opt/tomcat/tomcat.pid
        Environment=CATALINA_HOME=/opt/tomcat
        Environment=CATALINA_BASE=/opt/tomcat
        Environment="CATALINA_OPTS=-Xms512M -Xmx1024M -server -XX:+UseParallelGC"

        ExecStart=/opt/tomcat/bin/startup.sh
        ExecStop=/opt/tomcat/bin/shutdown.sh

        ExecReload=/bin/kill $MAINPID
        RemainAfterExit=yes

        [Install]
        WantedBy=multi-user.target 
        ```
    *   Enable and start:
        ```bash
        sudo systemctl daemon-reload
        sudo systemctl start tomcat
        sudo systemctl enable tomcat
        ```

4.  **Install CodeDeploy Agent (Ubuntu)**:
    ```bash
    sudo apt install ruby-full wget -y
    cd /home/ubuntu
    # Replace <YOUR_REGION> with your actual region, e.g., us-east-1
    wget https://aws-codedeploy-<YOUR_REGION>.s3.<YOUR_REGION>.amazonaws.com/latest/install
    chmod +x ./install
    sudo ./install auto
    sudo service codedeploy-agent status
    ```
5.  **IAM Role for EC2**:
    *   Attach a role with `AmazonEC2RoleforAWSCodeDeploy` and `AmazonS3ReadOnlyAccess` to your EC2.

---

## Step 1: Create CodeDeploy Application

*   **Application name**: `SnakesApp`.
*   **Deployment Group**: `SnakesDepGroup`.
*   **Service Role**: Use your CodeDeploy IAM Role.

---

## Step 2: Create a Build Project (CodeBuild)

*   **Project name**: `SnakesBuild`.
*   **Environment**:
    *   **OS**: `Ubuntu`.
    *   **Runtime**: `Standard`.
    *   **Image**: `standard:5.0` (or latest).
    *   **Service Role**: Select **"New service role"** (e.g., `codebuild-web-application-service-role`) - *Easiest option*.
    *   **VPC**: Select **No VPC** (Critical to avoid permissions errors).
*   **Buildspec**: Use the `buildspec.yml` in the source.

---

## Step 3: Create the Pipeline

*   **Source**: GitHub/CodeCommit.
*   **Build**: `SnakesBuild`.
*   **Deploy**: `SnakesApp` -> `SnakesDepGroup`.

---

## Troubleshooting

### Error: "Not authorized to perform DescribeSecurityGroups"
*   **Cause**: You selected a VPC in CodeBuild settings.
*   **Fix**: Edit CodeBuild Environment -> Additional Configuration -> Select **No VPC**.

### Error: "ScriptFailed" or Permission Denied
*   Ensure `appspec.yml` has the correct permissions (User `tomcat` needs to own the files). The updated `appspec.yml` in the repo now handles this.

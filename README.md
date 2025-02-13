# cicd_devops_Project


### CI/CD Pipeline to Deploy to Kubernetes Cluster Using Jenkins

## STEPS INVOVLED

- Install and Configure the Jenkins-Master and Jenkins-Agent
- Integrate Maven to Jenkins and Add GitHub Credentials to Jenkins
- Create Pipeline Script(Jenkinsfile) for Build & Test Artifacts and Create CI Job on Jenkins
- Install and Configure the SonarQube
- Integrate SonarQube with Jenkins
- Build and Push Docker Image using Pipeline Script
- Setup Bootstrap Server for eksctl and Setup Kubernetes using eksctl
- ArgoCD Installation on EKS Cluster and Add EKS Cluster to ArgoCD
- Configure ArgoCD to Deploy Pods on EKS and Automate ArgoCD Deployment Job using GitOps GitHub Repository
- DONE - Verify CI/CD Pipeline by Doing Test Commit on GitHub Application Repo

### 1. Set Up Jenkins Server

- Launch Jenkins Server:
- Use Amazon Linux 2 AMI.
- Instance type: t2.micro.
- Security Group: Open ports 8080 (Jenkins) and 22 (SSH).
- Attach 8GB storage.
- SSH into Jenkins Server.

## Install Jenkins:

```sh 
sudo yum update -y
sudo wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat-stable/jenkins.repo
sudo rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io-2023.key
sudo yum upgrade
sudo amazon-linux-extras install epel
sudo amazon-linux-extras install java-openjdk11 -y
sudo yum install java-11-amazon-corretto -y
sudo yum install jenkins -y
sudo systemctl enable jenkins
sudo systemctl start jenkins
java -version
javac -version
systemctl status jenkins
```

- Change Hostname:

```sh 
cd /etc
vim hostname
reboot
```

- Access Jenkins:
- Open browser: http://<public-ip>:8080.

## 2. Install and Configure Maven

- Download and Install Maven:

```sh 
sudo su
cd /opt
wget https://dlcdn.apache.org/maven/maven-3/3.9.3/binaries/apache-maven-3.9.3-bin.tar.gz
tar -xzvf apache-maven-3.9.3-bin.tar.gz
mv apache-maven-3.9.3 maven
cd maven/bin
./mvn -v
```

- Set Up Maven Path:

```sh 
cd ~
vim .bash_profile
```

- Add:

```sh 
M2_HOME=/opt/maven
M2=/opt/maven/bin
JAVA_HOME=/usr/lib/jvm/java-11-openjdk-11.0.19.0.7-1.amzn2.0.1.x86_64
PATH=$PATH:$HOME/bin:$JAVA_HOME:$M2_HOME:$M2
```

- Save and apply:

```sh 
source .bash_profile
echo $PATH
mvn -v
```

## 3. Configure Jenkins Plugins

- Install Plugins:
- Maven Integration.
- GitHub Plugin.
  
## Configure JDK and Maven in Jenkins:

- Go to Manage Jenkins > Tools.
- Add JDK: Name java11, Path to JAVA_HOME.
- Add Maven: Name maven, Path to /opt/maven.

## 4. Set Up Jenkins Job

- Create a New Maven Project:
- Source Code Management: Git Repo.
- Branch: */main.
- Root POM: pom.xml.
- Goals: clean install.
- Build the Project:
- Click Build Now.
- Artifact (WAR file) will be in job/workspace/webapp/target.

## Ansible Server Configuration

## 1. Launch an Amazon Linux 2 Instance
- Use the Amazon Linux 2 AMI.
- Select instance type: t2.micro.
- Configure Security Group:
- Open ports: 8080-8090 (for Jenkins) and 22 (for SSH).
- Launch the instance with at least 8GB storage.

## 2. SSH into the Server

- Use the following bash to connect:

```sh 
ssh -i your-key.pem ec2-user@your-public-ip
3. Configure the Server
```

- Set the hostname:

```sh 
sudo nano /etc/hostname
```

- Add your desired hostname and save the file.

- Create a new user (ansadmin):

```sh 
sudo useradd ansadmin
sudo passwd ansadmin
```

- Add ansadmin to the sudoers file:

```sh 
sudo visudo
```

- Add the following line:

```sh 
ansadmin ALL=(ALL) NOPASSWD: ALL
```

- Enable password authentication:

```sh 
sudo nano /etc/ssh/sshd_config
```

- Uncomment and set PasswordAuthentication yes.

- Reload the SSH service:

```sh 
sudo service sshd reload
```

- Switch to the ansadmin user:

```sh 
sudo su - ansadmin
```

- Generate SSH keys:

```sh 
ssh-keygen
```

- Press Enter to accept default settings.
- The public key will be saved at /home/ansadmin/.ssh/id_rsa.pub.

## 4. Install Ansible

- Switch to the root user:

```sh 
sudo su
```

- Install Ansible:

```sh 
amazon-linux-extras install ansible2
```

- Verify the installation:

```sh 
ansible --version
```

- Integrate Ansible with Jenkins

## 1. Configure Jenkins to Connect to Ansible

- Go to Manage Jenkins > System.
- Scroll down to Publish over SSH.
- Add a new SSH server:
- Name: ansible-server
- Hostname: Public IP of the Ansible server.
- Username: ansadmin
- Under Advanced:
- Enable Use password authentication.
- Provide the password for ansadmin.
- Click Test Configuration to verify the connection.
- Click Apply and Save.

## Summary

- Jenkins is configured to build a Maven project and generate a .war artifact.
- Ansible is installed and configured on an Amazon Linux 2 instance.
- Jenkins is integrated with Ansible for seamless deployment.

## 1.1 Create Docker Directory on Ansible Server

- SSH into the Ansible server:

```sh 
ssh -i your-key.pem ansadmin@your-ansible-server-ip
```

- Create a directory for Docker:

```sh 
cd /opt
sudo mkdir docker
sudo chown ansadmin:ansadmin docker
```

## 1.2 Configure Jenkins Job

- Go to the Jenkins Dashboard.
- Click New Item and select Maven Project.
- Configure the Git repository and branch (e.g., */main).
- Set the Root POM to pom.xml and add the goals: clean install.
- Under Post-build Actions, select Send build artifacts over SSH:
- SSH Server Name: ansible-server
- Source Files: webapp/target/*.war
- Remove Prefix: webapp/target
- Remote Directory: //opt//docker (use // to ensure the file is copied to the correct directory).
- Click Apply and Save.

## 1.3 Verify Artifact copy

- After the build, check the Ansible server:

```sh 
cd /opt/docker
ls
```

- Ensure the .war file is present.

## Step 2: Install and Configure Docker on Ansible Server

## 2.1 Install Docker

- Install Docker:

```sh 
sudo yum install docker -y
```

- Add ansadmin to the Docker group:

```sh 
sudo usermod -aG docker ansadmin
id ansadmin  # Verify the user is added to the Docker group
```

- Start Docker:

```sh 
sudo service docker start
sudo systemctl enable docker
sudo systemctl status docker
```

## 2.2 Reboot and Restart Docker

- Reboot the server:

```sh 
sudo reboot
```

- Reconnect and start Docker:

```sh 
sudo service docker start
```

## Step 3: Create Docker Image and Push to DockerHub

## 3.1 Create Dockerfile

- Navigate to the Docker directory:

```sh 
cd /opt/docker
```

- Create a Dockerfile:

```sh 
nano Dockerfile
```

- Add the following content:

Dockerfile

```sh 
FROM tomcat:latest
RUN cp -R /usr/local/tomcat/webapps.dist/* /usr/local/tomcat/webapps
copy  ./*.war /usr/local/tomcat/webapps
```

## 3.2 Build Docker Image

- Build the Docker image:

```sh 
docker build -t your-dockerhub-username/my-webapp:latest .
```

## 3.3 Push Image to DockerHub

- Log in to DockerHub:

```sh 
docker login
```

- Push the image:

```sh 
docker push your-dockerhub-username/my-webapp:latest
```

## Step 4: Create Ansible Playbook for Automation

## 4.1 Create Playbook

- Create a playbook file:

```sh 
nano docker-playbook.yml
```

- Add the following content:

yaml
```sh 
- hosts: localhost
  tasks:
    - name: Build Docker image
      bash: docker build -t your-dockerhub-username/my-webapp:latest /opt/docker

    - name: Push Docker image to DockerHub
      bash: docker push your-dockerhub-username/my-webapp:latest
```

## 4.2 Run the Playbook

- Execute the playbook:

```sh 
ansible-playbook docker-playbook.yml
```

## Summary

- Jenkins builds the Maven project and sends the .war artifact to the Ansible server.
- Docker is installed and configured on the Ansible server.
- A Docker image is created and pushed to DockerHub.
- Ansible automates the Docker image creation and push process.

## Prerequisites

- Before you begin, ensure you have:
- An AWS account to launch EC2 instances.
- Basic knowledge of Linux bashs.
- Security Group configured to allow ports 8080 (Jenkins), 22 (SSH), and 8080-8090 (Ansible).

## Step 1: Configure Ansible Hosts and SSH Key

## 1.1 Update Ansible Hosts File
- Open the Ansible hosts file:

```sh 
sudo nano /etc/ansible/hosts
```

- Delete everything and add the following:

```sh 
[ansible]
private-ip-address-of-ansible-server
```

## 1.2 Add SSH Key to Localhost

- copy  the SSH key to the Ansible server:

```sh 
ssh-copy -id ansadmin@private-ip-of-ansible-server
```

- Enter the password for ansadmin when prompted.

- Verify the key is copied:

```sh 
ssh ansadmin@private-ip-of-ansible-server
```

## Step 2: Create Ansible Playbook for Docker

## 2.1 Create Playbook File

- Navigate to the Docker directory:

```sh 
cd /opt/docker
```

- Create a playbook file:

```sh 
nano regapp.yml
```

- Add the following content:

yaml
```sh 
---
- hosts: ansible
  tasks:
    - name: Create Docker image
      bash: docker build -t regapp:latest .
      args:
        chdir: /opt/docker

    - name: Create tag to push image onto DockerHub
      bash: docker tag regapp:latest ashfaque9x/regapp:latest

    - name: Push Docker image
      bash: docker push ashfaque9x/regapp:latest
```

## 2.2 Run the Playbook

- Test the playbook:

```sh 
ansible-playbook regapp.yml --check
```

- Execute the playbook:

```sh 
ansible-playbook regapp.yml
```

## 2.3 Docker Login

- Log in to DockerHub:

```sh 
docker login
```

- Enter your DockerHub username and password.

## Step 3: Set Up Jenkins to Run Ansible Playbook
- Go to the Jenkins Dashboard.
- Configure the job to run the Ansible playbook:
- Add a Post-build Action to execute the playbook:

```sh 
ansible-playbook /opt/docker/regapp.yml
```

- Save and run the job.

## Step 4: Set Up Bootstrap Server for EKS

## 4.1 Launch EC2 Instance

- AMI: Ubuntu 22.04
- Instance Type: t2.micro
- Security Group: Allow inbound 22 (SSH)

## 4.2 Install AWS CLI

## SSH into the bootstrap server:

```sh 
ssh -i your-key.pem ubuntu@your-public-ip
```

- Install AWS CLI:

```sh 
sudo su
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
apt install unzip
unzip awscliv2.zip
sudo ./aws/install
```

## 4.3 Install kubectl

- Download and install kubectl:

```sh 
curl -O https://s3.us-west-2.amazonaws.com/amazon-eks/1.27.1/2023-04-19/bin/linux/amd64/kubectl
chmod +x ./kubectl
mv kubectl /bin
kubectl version --output=yaml
```

## 4.4 Install eksctl
- Download and install eksctl:

```sh 
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
cd /tmp
sudo mv /tmp/eksctl /bin
eksctl version
```

## 4.5 Create IAM Role

- Create an IAM role with EC2 Admin Access.
- Attach the role to the bootstrap server.

## 4.6 Set Up Kubernetes Cluster

- Create a Kubernetes cluster using eksctl:

```sh 
eksctl create cluster --name virtualtechbox-cluster \
  --region ap-south-1 \
  --node-type t2.small \
  --nodes 3
```

- This process may take up to 30 minutes.

## Summary

- Jenkins builds the Maven project and triggers Ansible to create and push Docker images.
- Ansible automates Docker image creation and pushes them to DockerHub.
- A bootstrap server is set up to manage Kubernetes clusters using eksctl.




## ArgoCD Installation and EKS Cluster Integration

- This guide provides step-by-step instructions to install ArgoCD on a Kubernetes cluster, expose it using a LoadBalancer, and integrate an EKS cluster with ArgoCD.

## Table of Contents

- Install ArgoCD
- Expose ArgoCD Server
- Access ArgoCD Dashboard
- Add EKS Cluster to ArgoCD
- Install ArgoCD

## 1. Create ArgoCD Namespace
- Create a dedicated namespace for ArgoCD:

```sh 
kubectl create namespace argocd
```

## 2. Apply ArgoCD Manifest

- Install ArgoCD using the official manifest:

```sh 
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

## 3. Verify Pods

- Check if all ArgoCD pods are running:

```sh 
kubectl get pods -n argocd
```

## 4. Install ArgoCD CLI

- Download and install the ArgoCD CLI:

```sh 
curl --silent --location -o /usr/local/bin/argocd https://github.com/argoproj/argo-cd/releases/download/v2.4.7/argocd-linux-amd64
chmod +x /usr/local/bin/argocd
```

- Expose ArgoCD Server

## 1. Patch Service to Use LoadBalancer

- Expose the ArgoCD server to the external world:

```sh 
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
```

## 2. Get LoadBalancer URL

Retrieve the LoadBalancer URL:

```sh 
kubectl get svc -n argocd
```

- copy the EXTERNAL-IP or DNS name.
- Access ArgoCD Dashboard

## 1. Retrieve Admin Password

- Extract the initial admin password:

```sh 
kubectl get secret argocd-initial-admin-secret -n argocd -o yaml
```

- Decode the password:

```sh 
echo <base64-encoded-password> | base64 --decode
```

## 2. Log in to ArgoCD Dashboard

- Open the LoadBalancer URL in your browser.
- Log in with:
- Username: admin
- Password: The decoded password.

## 3. Update Password

- Go to User Info in the ArgoCD dashboard.
- Update the password for security.

## Add EKS Cluster to ArgoCD

## 1. Log in to ArgoCD CLI
- Log in to ArgoCD using the CLI:

```sh 
argocd login <argocd-loadbalancer-url> --username admin
```

- Enter the password when prompted.

## 2. List Existing Clusters
- View the default cluster:

```sh 
argocd cluster list
```

## 3. Add EKS Cluster

- Get the EKS cluster context:

```sh 
kubectl config get-contexts
```

- Add the EKS cluster to ArgoCD:

```sh 
argocd cluster add <context-name> --name <cluster-name>
```

Example:

```sh 
argocd cluster add i-08b9d0ff0409f48e7@virtualtechbox-cluster.ap-south-1.eksctl.io --name virtualtechbox-eks-cluster
```

## 4. Verify Cluster Addition

- List clusters to confirm the EKS cluster is added:

```sh 
argocd cluster list
```

## 5. Check ArgoCD Dashboard

- Go to the ArgoCD dashboard.
- Navigate to Settings > Clusters.
- Verify the newly added EKS cluster.

## Summary

- ArgoCD is installed and exposed using a LoadBalancer.
- The EKS cluster is integrated with ArgoCD for continuous delivery.
- You can now manage deployments to the EKS cluster using ArgoCD.


## Configure ArgoCD to Deploy Pods on EKS and Automate ArgoCD Deployment Job using GitOps

## This guide provides step-by-step instructions to:

- Connect a GitHub repository with Kubernetes manifests to ArgoCD.
- Deploy resources on an EKS cluster using ArgoCD.
- Automate the deployment process using Jenkins and GitOps.

## Table of Contents

- Connect GitHub Repository to ArgoCD
- Deploy Resources on EKS Cluster
- Verify Deployment
- Automate ArgoCD Deployment Job using Jenkins

## Connect GitHub Repository to ArgoCD
## 1. Add GitHub Repository to ArgoCD

- Go to the ArgoCD Dashboard.
- Navigate to Settings > Repositories.
- Click Connect Repo:
- Type: Git
- Project: Default
- Repository URL: GitHub repository URL (without .git)
- Username: Your GitHub username
- Password: GitHub Personal Access Token
- Click Connect.

## Deploy Resources on EKS Cluster

## 1. Create a New Application in ArgoCD

## Go to the ArgoCD Dashboard.

- Click New App.
- Fill in the details:
- Application Name: register-app
- Project Name: Default
- Sync Policy: Automatic (enable Prune Resource and Self-Heal)

## Under Source:

- Repository URL: GitHub repository URL
- Revision: HEAD
- Path: ./ (path to the manifest files in the repository)

## Under Destination:

- Cluster URL: EKS cluster URL
- Namespace: Default
- Create.

## 2. Verify Deployment

- Go to the Bootstrap Server.
- Check the deployed pods:

```sh 
kubectl get pods
```

- Get the external DNS name of the LoadBalancer:

```sh 
kubectl get svc
```

- copy the EXTERNAL-IP or DNS name and browse it (e.g., http://<external-dns>:8080).
- Verify the application is running.

### Automate ArgoCD Deployment Job using Jenkins

## 1. Create a New Jenkins Job

- Go to the Jenkins Dashboard.
- Click New Item.
- Enter a name (e.g., register-app-cd-job) and select Pipeline.
- Click OK.

## 2. Configure the Jenkins Job

## Under General:

- Enable Discard old builds.
- Set Max # of builds to keep to 2.
- Enable This project is parameterized.
- dd a String Parameter named IMAGE_TAG.

## Under Build Triggers:

- Enable Trigger builds remotely.
- Provide a token (e.g., gitops-token).
- Under Pipeline:

## Select Pipeline script from SCM.

- SCM: Git
- Repository URL: GitHub repository URL
- Credentials: Add your GitHub credentials.
- Branch: main
- Click Apply and Save.

## 3. Trigger the Jenkins 

- Manually trigger the job by providing the IMAGE_TAG parameter.
- Alternatively, configure a webhook in GitHub to trigger the job automatically on code changes.

## Summary

-  ArgoCD is configured to deploy resources on the EKS cluster using manifests from a GitHub repository.
-  Jenkins automates the deployment process, ensuring continuous delivery using GitOps principles.

### Automating CI/CD Pipeline with Jenkins and ArgoCD

- This guide provides step-by-step instructions to:
- Set up a Jenkins pipeline to trigger a CD pipeline.
- Automate the deployment process using ArgoCD.
- Verify the CI/CD pipeline by making test commits to the GitHub repository.

### Table of Contents

- Trigger CD Pipeline from Jenkins
- Jenkins Pipeline Script
- Verify CI/CD Pipeline
- Trigger CD Pipeline from Jenkins

## 1. Jenkins Pipeline Stage to Trigger CD Pipeline

- Add the following stage to your Jenkins pipeline script to trigger the CD pipeline:

groovy
```sh 
stage("Trigger CD Pipeline") {
    steps {
        script {
            sh """
                curl -v -k --user clouduser:${JENKINS_API_TOKEN} -X POST \
                -H 'cache-control: no-cache' \
                -H 'content-type: application/x-www-form-urlencoded' \
                --data 'IMAGE_TAG=${IMAGE_TAG}' \
                'http://ec2-13-232-128-192.ap-south-1.compute.amazonaws.com:8080/job/gitops-register-app-cd/buildWithParameters?token=gitops-token'
        }   """
        
    }
}

```

- IMAGE_TAG: The Docker image tag to be deployed.
- JENKINS_API_TOKEN: Jenkins API token for authentication.
- gitops-token: Token provided in the Jenkins CD job.

## 2. Generate Jenkins API Token

- Go to the Jenkins Dashboard.
- Navigate to your username > Configure.
- Under API Token, click Add New Token.
- Name the token (e.g., jenkins_api_token) and generate it.
- copy the plain text token.

## 3. Add Jenkins API Token to Credentials

- Go to Manage Jenkins > Credentials.
- Add a new credential:
- Kind: Secret text
- Secret: Paste the Jenkins API token.
- ID: jenkins_api_token
- Save the credential.

## 4. Define Jenkins API Token in Pipeline Script

- Add the token as an environment variable in your Jenkins pipeline script:

groovy
```sh 
environment {
    JENKINS_API_TOKEN = credentials('jenkins_api_token')
}
```

#### Jenkins Pipeline Script for the CI/CD process:

- Verify CI/CD Pipeline

## 1. Set Up Triggers
- Go to the Jenkins CI Job.
- Under Build Triggers, enable Poll SCM.
- Set the schedule to poll every minute:

```sh 
* * * * *
```

## 2. Make a Test Commit

- Make a change to the application code in the GitHub repository.
- Push the changes to the main branch.

## 3. Verify Pipeline Execution

- The CI Job will trigger automatically.
- Once the CI Job completes, the CD Job will trigger automatically.
  
- Verify the deployment:
- Check the Docker image tag in DockerHub (it should be updated to the latest version).
- Check the deployment.yaml file in the GitHub repository (the image tag should be updated).
- Go to the ArgoCD Dashboard and verify the application is using the latest release.
- Check the pods in the EKS cluster:


```sh 
kubectl get pods
```

## Summary

- The Jenkins pipeline automates the CI/CD process, including updating the deployment manifest and triggering the CD pipeline.
- ArgoCD ensures the latest version of the application is deployed to the EKS cluster.
- The entire process is triggered automatically by changes to the GitHub repository.




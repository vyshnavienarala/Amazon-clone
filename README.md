# Amazon Clone Website with CI/CD Pipeline

## Aim
The aim of this project is to build and deploy an Amazon clone website with an automated CI/CD pipeline. This pipeline, integrated with GitHub, is triggered by a webhook and uses Jenkins for continuous integration and Docker for containerized deployment. The website is hosted on an NGINX web server running on an AWS EC2 instance.

## Components
- **EC2 Instance**: Hosts the Jenkins server and NGINX for the web application.
- **Jenkins**: Automates code builds and deployment.
- **GitHub**: Stores the source code and triggers Jenkins via webhooks.
- **Docker**: Manages the containerization of the web application.
- **NGINX**: Acts as the web server for the deployed application.
- **SSH Keys**: Used for secure communication between Jenkins and GitHub.

## Problem Definition
The challenge is to automate the process of deploying the Amazon clone website. Each time changes are committed to the GitHub repository, Jenkins should automatically build the application and deploy it using Docker on the EC2 instance. The solution involves setting up a secure and efficient pipeline that connects Jenkins, GitHub, and Docker.

## Architecture

![Build and Deploy an Amazon Clone Website with Automated CICD Pipeline Using Jenkins and GitHub](https://github.com/user-attachments/assets/8b0c6bd8-d222-49c3-8742-f6fa6e1a720d)

## Steps

### 1. **Create an EC2 Instance**
- **Instance Name**: Amazon Clone
- **AMI**: Ubuntu 24.04
- **Instance Type**: t2.micro
- **Security Group**: Enable:
  - HTTP
  - HTTPS
  - Custom TCP (8080, Anywhere)
  - Custom TCP (8000, Anywhere)
- **Action**: Launch the instance.

### 2. **Connect to the EC2 Instance and Install Required Packages**

Run the following commands in the terminal after connecting via SSH:

```bash
# Update the package list
sudo apt update

# Install OpenJDK 17
sudo apt install openjdk-17-jdk

# Verify the Java installation
java -version

# Add the Jenkins GPG key
sudo wget -O /usr/share/keyrings/jenkins-keyring.asc https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key

# Add Jenkins repository to the sources list
echo "deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/" | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null

# Update the package list
sudo apt-get update

# Install Jenkins
sudo apt-get install jenkins

# Enable Jenkins to start on boot and start the service
sudo systemctl enable jenkins
sudo systemctl start jenkins

# Check Jenkins status
sudo systemctl status jenkins

# Install and start Nginx
sudo apt install nginx -y
sudo service nginx start

# Enable Nginx to start on boot
sudo service nginx enable

# Check Nginx status
sudo service nginx status
```

### 3. Access Jenkins
- Copy the **Public IP Address** of the EC2 instance and paste it into a browser followed by `:8080`.  
  - Example: `http://43.205.241.55:8080`
- Use the **initialAdminPassword** located in `/var/lib/jenkins/secrets/initialAdminPassword`.
- Follow the Jenkins setup steps:
  1. Click **Install Suggested Plugins**.
  2. Create a username, password, and provide email details.
  3. Click **Save & Continue** and **Start Using Jenkins**.

### 4. Create GitHub Personal Access Token
1. Go to **GitHub Profile** → **Settings** → **Developer Settings** → **Personal Access Tokens (classic)**.
2. Generate a new token:
   - **Note**: Jenkins-CICD
   - **Expiration**: 30 days
   - **Scopes**: Enable `repo` and `workflow`.
3. Copy the **Personal Access Token**.

### 5. Connect Jenkins to GitHub
- In Jenkins:
  1. Navigate to **Manage Jenkins** → **Configure System**.
  2. Scroll down to **GitHub Server**:
     - **Name**: GitHub
     - **API URL**: `https://api.github.com`
     - **Credentials**: Add the Personal Access Token.
       - **Kind**: Secret Text
       - **Secret**: (Paste the personal access token).
       - **ID**: Jenkins-For-CICD.
  3. Test the connection to see your GitHub username and **Save**.

### 6. Create a Jenkins Job
1. In Jenkins, create a new item:
   - **Item Name**: `kdm-amazon-clone`
   - **Type**: Freestyle Project
   - **General**: Enable **GitHub Project** and provide the repository URL.
2. Under **Source Code Management**:
   - **Repository URL**: `https://github.com/KavitDeepakMehta/AMAZON-CLONE`
   - **Branch Specifier**: `*/main`
3. **Build Steps**:  
   Add a build step: **Execute Shell** and enter the following commands:
   
    ```bash
    sudo docker ps --filter "publish=8000" -q | xargs -r docker rm -f
    sudo docker build . -t kdm-amazon-clone
    sudo docker run -p 8000:80 -d kdm-amazon-clone
    ```
    **NOTE**: Before Clicking Build Now First Make Changes in "sudo visudo"
   #### The sudo visudo command opens and edits the sudoers file, which is a configuration file used by the sudo program to control which users can run commands as root or other users.

   ```bash
    # Open the sudoers file for editing
    sudo visudo
    
    # Add this at the end in the file to allow the Jenkins user to run Docker commands without a password:
    jenkins ALL=(ALL) NOPASSWD: /usr/bin/docker
    
    # List running Docker containers again
    sudo docker ps
    
    # Kill a specific Docker container (replace with your container ID)
    sudo docker kill 233a2df5f25d
   ```
4. Save and click **Build Now**.  
   After success, access the web app via `http://43.205.241.55:8000`.

### 7. Automate Deployment Using SSH Keys and Webhooks
1. Generate SSH keys:
   
    ```bash
    # Generate SSH keys
    ssh-keygen

    # Navigate to the .ssh directory
    cd .ssh

    # List all files
    ls -al

    # View public SSH key
    cat ~/.ssh/id_ed25519.pub

    # View private SSH key (use cautiously)
    cat ~/.ssh/id_ed25519
    ```

2. Copy the **public key** and add it to **GitHub**:
   - Go to **Settings** → **SSH and GPG Keys** → **New SSH Key**.
   - **Title**: Jenkins Public SSH Key.
   - **Key Type**: Authentication Key.
   - **Key**: (Paste the public key).

3. In Jenkins, add the private SSH key:
   - Go to **Manage Jenkins** → **Configure System** → **GitHub Server**:
     - **Kind**: SSH Username and Private Key.
     - **Private Key**: Enter the private SSH key manually.

### 8. Set Up GitHub Webhooks
- In GitHub:
  1. Go to **Repository Settings** → **Webhooks** → **Add Webhook**.
  2. **Payload URL**: `http://43.205.241.55:8080/github-webhook/`
  3. **Content Type**: `application/json`.
  4. Add the webhook.

- In Jenkins:
  1. Go to the **kdm-amazon-clone Job**.
  2. Under **Build Triggers**, enable **GitHub hook trigger for GITScm polling**.

### 9. Grant Docker Permissions to Jenkins
Run the following commands to grant Jenkins permission to run Docker:

```bash
# Add Jenkins user to the Docker group
sudo usermod -aG docker jenkins

# Restart Jenkins and Docker
sudo systemctl restart jenkins
sudo systemctl restart docker

# Verify group membership
sudo su - jenkins
groups # Should see "jenkins docker"
exit
```

### 10. Test Automatic Builds
- Make a change in the **GitHub Repository** (e.g., modify `index.html`) and commit it.
- Jenkins will automatically trigger the build process, and the web application should reflect the changes in real-time.

## Documentation Link
https://docs.google.com/document/d/1qXoAuRETI6su_ZW7uryk-DFNLuU_waWyZ8BXUO_NPIc/edit?usp=sharing

## Implementation Video Link
https://github.com/user-attachments/assets/1aa25aa4-e854-4703-b0c4-ee0ab641e470

## Expected Outcome
By the end of this process, the Amazon clone website will be automatically deployed to an EC2 instance using a Docker container. Jenkins will continuously integrate new changes from the GitHub repository, triggering builds and deployments automatically whenever a new commit is made. The final web application will be accessible via the EC2 public IP with port 8000.

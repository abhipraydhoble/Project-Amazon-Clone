
| project amazon-app |
|--------|
| <img width="275" height="183" src="https://github.com/user-attachments/assets/17b4d5fc-3b39-4fed-894e-8fa3e5911784" /> |



## Tech Stack:

- Terraform (Infrastructure as Code)
- AWS EC2
- Jenkins CI/CD
- SonarQube Quality Analysis
- Docker Image Build & Push
- Trivy Image Scan
- OWASP Dependency Check
- Node.js + JDK17 Environment



---

## 📌 1. Install Required Tools (Ubuntu)

### Install AWS CLI
```bash
sudo apt update
sudo apt install awscli -y
```

### Install Terraform
```bash
sudo apt-get update && sudo apt-get install -y gnupg software-properties-common
wget -O- https://apt.releases.hashicorp.com/gpg | gpg --dearmor | sudo tee /usr/share/keyrings/hashicorp.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update
sudo apt install terraform -y
```

---

## 📌 2. Configure AWS CLI
Run:
```bash
aws configure
```
Enter:
- Access Key
- Secret Key
- Region
- Output: json

---

## 📌 3. Clone the Repository
```bash
mkdir amazon
cd amazon
git clone https://github.com/roshand33/Project-Amazon-Clone.git
ls
```

Move into Terraform folder:
```bash
cd Project-Amazon-Clone/JENKINS-TF
```

---

## 📌 4. Update Terraform Files

### provider.tf
Set your AWS region:
```hcl
region = "ap-south-1"
```

### main.tf
Update:
- ami
- instance_type (ex: t2.micro, t3.medium)

---

## 📌 5. Deploy Infrastructure
```bash
terraform init
terraform validate
terraform plan
terraform apply --auto-approve
```

This creates:
- Jenkins EC2 Server
- Security Groups
- Required networking

---

## 📌 6. Login to Jenkins Server

Get EC2 Public IP → open browser:
```
http://<PUBLIC-IP>:8080
```

Get Jenkins admin password:
```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

Paste into browser → login.

---

## 📌 7. Install Required Jenkins Plugins

Install:

1. Eclipse Temurin Installer  
2. SonarQube Scanner  
3. NodeJS Plugin  
4. OWASP Dependency Check  
5. Prometheus Metrics  
6. Docker (ALL options + CloudBees)  
7. Stage View  

---

## 📌 8. Configure Credentials in Jenkins

## Sonar Token
Open Sonar:
```
http://<instance-ip>:9000
```

Go to:
Administration → Security → Tokens  
Generate token: `sonar-token`

Add in Jenkins:
- Manage Jenkins → Credentials
- Add Secret Text
- ID: sonar-token

---

## Docker Hub Credentials
Add:
- Username
- Password
- ID: docker

---

## 📌 9. Configure Tools (Manage Jenkins → Tools)

### JDK
- Name: jdk17
- Install from adoptium
- Version: 17

### NodeJS
- Name: node16
- Version: 16.2.0

### Sonar Scanner
- Default settings

### Docker
- Install via Jenkins installer

---

## 📌 10. SonarQube Integration in Jenkins
Manage Jenkins → System  
Add:
- SonarQube Server
- Token ID: sonar-token

---

## 📌 11. Jenkins Pipeline Script

Replace `<YOUR-DOCKER-USERNAME>` with your Docker Hub username.

```groovy
pipeline{
    agent any
    tools{
        jdk 'jdk17'
        nodejs 'node16'
    }
    environment {
        SCANNER_HOME=tool 'sonar-scanner'
    }
    stages {
        stage('clean workspace'){
            steps{
                cleanWs()
            }
        }
        stage('Checkout from Git'){
            steps{
                git branch: 'main', url: 'https://github.com/roshand33/Project-Amazon-Clone.git'
            }
        }
        stage("Sonarqube Analysis "){
            steps{
                withSonarQubeEnv('sonar-server') {
                    sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=Amazon \
                    -Dsonar.projectKey=Amazon '''
                }
            }
        }
        stage("quality gate"){
           steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'jenkins' 
                }
            } 
        }
        stage('Install Dependencies') {
            steps {
                sh "npm install"
            }
        }
        stage('OWASP FS SCAN') {
            steps {
                dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit', odcInstallation: 'DP-Check'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }
        stage('TRIVY FS SCAN') {
            steps {
                sh "trivy fs . > trivyfs.txt"
            }
        }
        stage("Docker Build & Push"){
            steps{
                script{
                   withDockerRegistry(credentialsId: 'docker', toolName: 'docker'){   
                       sh "docker build -t amazon-clone ."
                       sh "docker tag amazon-clone <YOUR-DOCKER-USERNAME>/amazon-clone:latest"
                       sh "docker push <YOUR-DOCKER-USERNAME>/amazon-clone:latest"
                    }
                }
            }
        }
        stage("TRIVY"){
            steps{
                sh "trivy image <YOUR-DOCKER-USERNAME>/amazon-clone:latest > trivyimage.txt"
            }
        }
        stage('Deploy to container'){
            steps{
                sh 'docker run -d --name amazon-clone -p 3000:3000 <YOUR-DOCKER-USERNAME>/amazon-clone:latest'
            }
        }
    }
}
```

---

# ⚠️ Mandatory Changes Before Running

1. Change Docker Hub username  
2. Optional: fork repo and replace Git URL  
3. Verify names:  
   - sonar-server  
   - sonar-token  
   - sonar-scanner  

---

## 📌 12. Restart Jenkins
```
/restart
```

---

## 📌 13. Run the Pipeline

Click **Build Now**  
Wait **10–50 minutes**  
Pipeline will run:
- Git Checkout  
- Sonar Analysis  
- Quality Gate  
- OWASP Scan  
- Trivy Scan  
- Docker Build & Push  
- Deployment  

---

---


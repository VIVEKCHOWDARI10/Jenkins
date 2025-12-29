This is my way of executing the project with some explanation


## Check if AWS is installed in your UTM ,if not : 
``` bash 
aws --version
curl "https://awscli.amazonaws.com/awscli-exe-linux-aarch64.zip" -o "awscliv2.zip"
sudo yum install unzip -y
unzip awscliv2.zip
sudo ./aws/install
aws --version

```

## configure AWS:
``` bash
aws configure
```
Enter:
AWS Access Key ID: xxxx

AWS Secret Access Key: xxxx

Region: ap-south-1

Output format: json




Download the eksctl :
```bash
curl -LO https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_Linux_arm64.tar.gz
```


 Extract it 
``` bash 
tar -xzf eksctl_Linux_arm64.tar.gz
```

move to the system path :
```bash 
sudo mv eksctl /usr/local/bin/
```
# Now create a EKS cluster 
``` bash 
eksctl create cluster \
--name jenkins \
--region ap-south-1 \
--zones ap-south-1a,ap-south-1b,ap-south-1c \
--nodes 1
```

This will create a new ec2  instance  as Kubernetes  node ,name =Jenkins-ng-77xxx something as a worker node as u mentioned 1 worker node 
``` bash 
aws eks update-kubeconfig --name jenkins --region=ap-south-1
```
  
Reason: aws eks update-kubeconfig --name <cluster> updates the local kubeconfig so kubectl can connect to the EKS cluster with proper credentials and endpoint

## Create EC2 instance :
  Ssh into the ec2 using .pem
  ``` bash 
  Cd Downloads 
  Chmod 400 keypair.pem 
  ssh -i  keypair.pem ubuntu@ec2-ip
```

Make the ssh stay long time without logged out:
``` bash 
Vi ~/.ssh/config
```

Add this :
``` bash 
Host *
  ServerAliveInterval 30
  ServerAliveCountMax 120
```

If u want to change to the root user type   
``` bash
sudo su -
exit 
```


Install java and jenkins in the ec2 instance 


# Install Java (JDK 17)
``` bash 
sudo su -
apt update -y
apt install openjdk-17-jre -y
java -version
```

# Install Jenkins
``` bash 
curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key | sudo tee \
  /usr/share/keyrings/jenkins-keyring.asc > /dev/null

echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null
```

``` bash 
 apt update -y
apt install jenkins -y
```
 ## This will install Jenkins as a separate user in the ec2  machine 
``` bash 
apt update -y
sudo systemctl start jenkins
sudo systemctl enable jenkins
sudo systemctl status jenkin
```

Access the Jenkins intial code at :       
``` bash
cat /var/lib/jenkins/secrets/initialAdminPassword 
```


# Install Git
``` bash 
apt install git -y
git --version
```



# Install Docker & add jenkins to Docker group
``` bash 
apt update -y
apt install docker.io -y
usermod -aG docker jenkins
```


# Install maven
``` bash 
apt install maven
mvn -version
```

# Add these to .bashrc
``` bash 
export JAVA_HOME=$(dirname $(dirname $(readlink -f $(which java)))) 
export MAVEN_HOME=/usr/share/maven
export PATH=$PATH:$MAVEN_HOME/bin                        
source ~/.bashrc
```


👉 Login to the Jenkins and install docker pipeline and maven integration plugin

👉 Go to tools and add maven and give the path to it (that we downloaded before ), Type mvn —version there you can find the path 

                                    
You can directly add maven and its version in Jenkins, add maven option will download that 



Now  create a pipeline and add the script normally but take only checkout stage ,here u need to configure the Jenkins and git (I mean connectivity  between them ). 
## create a key pair and store the public key in the git  and private key in the jenkins 


## Jenkins-Github Integration
``` bash 
sudo -u jenkins ssh-keygen -t rsa -b 4096 -C "vivekchowdari10@gmail.com"
```


👉 Creates an SSH key pair:

* Private key → stays on Jenkins
  
* Public key → shared with GitHub / server

We generate an SSH key under the Jenkins user so Jenkins can securely authenticate to GitHub or servers without human intervention in CI/CD pipelines.

For production-level DevOps:
✅ SSH keys



HOW TOKEN METHOD WORKS (SIMPLE)
1️⃣ GitHub → Generate token 2️⃣ Jenkins → Store token securely 3️⃣ Jenkins uses token when running git clone
No SSH keys needed.


# Here you can  get the public key 
``` bash 
Su - jenkins 
Cat /var/lib/jenkins/.ssh/id_rsa.pub
``` 

Go to the GitHub and add the public key in the GitHub -> settings -> ssh and gpg keys  -> new shh key ->  add the key 
   

👉 here u get the private key ,Go to the jenkins and add the private  key in the credentials global  

kind= ssh username with private key   and add key at private -> direct add
``` bash
cat /var/lib/jenkins/.ssh/id_rsa
```


Now add the first part of the the pipeline and build it we will get the error  so go the jenkins user of ec2 and clone the repo normally so it  will add the gihub id to the host so now if u want u can delete the file u got and again start building , now it will work

``` bash 
Cd /tmp 
git clone git@github.com:jmbharathram/SampleJavaApps.git
rm  -rf SampleJavaApps
```
``` bash 
pipeline {
    agent any
    stages {
        stage('Checkout') {
            steps {
                git url: 'https://github.com/VIVEKCHOWDARI10/SampleJavaApps.git',
                    branch: 'main',
                    credentialsId: 'git-ssh-cred'
                dir('MyJenkins') {
                    git url: 'https://github.com/VIVEKCHOWDARI10/SampleJavaApps.git',
                        branch: 'main',
                        credentialsId: 'git-ssh-cred'
                }
            }
        }
    }
}

```

Now add the build step :


``` bash 

    pipeline {
    agent any
    stages {
        stage('Checkout') {
            steps {
                git url: 'https://github.com/VIVEKCHOWDARI10/SampleJavaApps.git',
                    branch: 'main',
                    credentialsId: 'git-ssh-cred'
                dir('MyJenkins') {
                    git url: 'https://github.com/VIVEKCHOWDARI10/SampleJavaApps.git',
                        branch: 'main',
                        credentialsId: 'git-ssh-cred'
                }
            }
        }
        stage('Build') {
            steps {
               sh 'ls -ltr'
               sh 'hostname'
               sh 'cd SpringAPI && mvn clean package'
            }
        }
    }
}

```

👉Now we have .jar file In the /target/

👉 Now add another step but In this  Jenkins try to connect to the docker so u need to add the credentials (username and password of your docker hub)  With  docker-secret as id ,as we used this in code


👉 Before building it ,switch to the root user in the ec2 and restart the jenkins (Important)
Now build it again if we ge the image error ,change the image and also the docker image name 


######## skip:


FROM eclipse-temurin:17-jdk

DOCKER_IMAGE = “vivekchowdari/springapi:${BUILD_NUMBER}

##############


Don’t forgot to restart jenkins as we installed couple of things 
``` bash 
Sudo systemctl restart jenkins
```
``` bash 
pipeline {
    agent any
    stages {
        stage('Checkout') {
            steps {
                git url: 'https://github.com/VIVEKCHOWDARI10/SampleJavaApps.git',
                    branch: 'main',
                    credentialsId: 'git-ssh-cred'
                dir('MyJenkins') {
                    git url: 'https://github.com/VIVEKCHOWDARI10/SampleJavaApps.git',
                        branch: 'main',
                        credentialsId: 'git-ssh-cred'
                }
            }
        }
        stage('Build') {
            steps {
               sh 'ls -ltr'
               sh 'hostname'
               sh 'cd SpringAPI && mvn clean package'
            }
        }
        stage('Build and Push Docker Image') {
            environment {
              DOCKER_IMAGE = "vivekchowdari/springapi:${BUILD_NUMBER}"
              REGISTRY_CREDENTIALS = credentials('docker-secret')
            }
            steps {
              script {
                  sh 'cd SpringAPI && docker build -t ${DOCKER_IMAGE} .'
                  def dockerImage = docker.image("${DOCKER_IMAGE}")
                  docker.withRegistry('https://index.docker.io/v1/', "docker-secret") {
                      dockerImage.push()
                  }
              }
            }
        }
    }
}

```

Now we are going to add the final stage in the pipeline script that is updating the image deployment yaml file  ,which is in the  another repo not in the same repo of code 



👉 Before doing that install the argocd in the kubernetees 


Go to the normal utm terminal because u downloaded the Kubernetes there :
``` bash 
aws eks update-kubeconfig --name jenkins --region=ap-south-1
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl get pods -n argocd
```

👉 port forward the argocd service to access its UI
``` bash 
kubectl port-forward svc/argocd-server -n argocd 8081:443
```
``` bash 
https://Localhost:8081
```


👉
👉 While port forwarding u should not stop the command because it is running in the foreground so if u stop u cant access  the argocd ui



 To get the username and password  of argocd :
 
Username = admin 

# To retrieve initial admin password
``` bash 
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```




👉 Now create a application in argocd for deloyment git hub link and  add path to the springapi_application.yaml  file 

👉All pods are automatically created 
``` bash 
Kubectl get pods 
kubectl get service
```



Now go to the jenkins pipeline and add final stage 

``` bash 

pipeline {
    agent any
    stages {
        stage('Checkout') {
            steps {
                git url: 'https://github.com/VIVEKCHOWDARI10/SampleJavaApps.git',
                    branch: 'main',
                    credentialsId: 'git-ssh-cred'
                dir('MyJenkins') {
                    git url: 'https://github.com/VIVEKCHOWDARI10/Jenkins.git',
                        branch: 'main',
                        credentialsId: 'git-ssh-cred'
                }
            }
        }
         stage('Build') {
            steps {
               sh 'ls -ltr'
               sh 'hostname'
               sh 'cd SpringAPI && mvn clean package'
            }
        }
      stage('Build and Push Docker Image') {
            environment {
              DOCKER_IMAGE = "vivekchowdari/springapi:${BUILD_NUMBER}"
              REGISTRY_CREDENTIALS = credentials('docker-secret')
            }
            steps {
              script {
                  sh 'cd /var/lib/jenkins/workspace/jenkins-ci-cd/SpringAPI && docker build -t ${DOCKER_IMAGE} .'
                  def dockerImage = docker.image("${DOCKER_IMAGE}")
                  docker.withRegistry('https://index.docker.io/v1/', "docker-secret") {
                      dockerImage.push()
                  }
              }
            }
        }
    stage('Update Image') {
            steps {
                script {
                    sh """
                    git config --global user.email "vivekchowdari10@gmail.com"
                    git config --global user.name "VIVEKCHOWDARI10"
                    cd   MyJenkins/Manifests/
                    sed -i 's,vivekchowdari/springapi.*,vivekchowdari/springapi:${BUILD_NUMBER},g' springapi_application.yaml
                    cat springapi_application.yaml
                    git add springapi_application.yaml
                    git commit -m "Image updated by Jenkins CI pipeline"
                    git push git@github.com:VIVEKCHOWDARI10/Jenkins.git main
                    """
                }
            }
        }
    }
}

```


👉Go to the utm :
``` bash 
Kubectl get services
``` 

👉 copy the  external ip of the springapi-service 


## ACCESS THE APPLICATION AT  :
http://aa9fba066c46c40cfa8bb952db290218-750121722.ap-south-1.elb.amazonaws.com/user?id=1

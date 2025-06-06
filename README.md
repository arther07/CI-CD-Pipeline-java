# **CI/CD Pipeline with Jenkins, Docker, and Kubernetes**

## **Overview**
This project demonstrates a **CI/CD pipeline** using **Jenkins, Docker, and Kubernetes** to build, test, and deploy  **Spring Boot application**.

## **Pipeline Workflow**
1. **Checkout Code** from GitHub.
2. **Build & Test** the application using Maven.
3. **Build & Push Docker Image** to Docker Hub.
4. **Update Kubernetes Deployment File** with the new image version.
5. **Deploy the Application** in a Kubernetes Cluster

---

## **Jenkins Pipeline (`Jenkinsfile`)**

```groovy
pipeline {
  agent {
    docker {
      image 'abhishekf5/maven-abhishek-docker-agent:v1'
      args '--user root -v /var/run/docker.sock:/var/run/docker.sock'
    }
  }
  stages {
    stage('Checkout') {
      steps {
        sh 'echo passed'
        //git branch: 'main', url: 'https://github.com/puneethsgit/Java-Applcation-CI-CD'
      }
    }
    stage('Build and Test') {
      steps {
        sh 'ls -ltr'
        sh 'cd spring-boot-app && mvn clean package'
      }
    }
    stage('Build and Push Docker Image') {
      environment {
        DOCKER_IMAGE = "puneeth11/newultimate-cicd-pipeline:${BUILD_NUMBER}"
        REGISTRY_CREDENTIALS = credentials('docker-cred')
      }
      steps {
        script {
            sh 'cd spring-boot-app && docker build -t ${DOCKER_IMAGE} .'
            def dockerImage = docker.image("${DOCKER_IMAGE}")
            docker.withRegistry('https://index.docker.io/v1/', "docker-cred") {
                dockerImage.push()
            }
        }
      }
    }
    stage('Update Deployment File') {
      environment {
        GIT_REPO_NAME = "Java-Applcation-CI-CD"
        GIT_USER_NAME = "arther07git"
      }
      steps {
        withCredentials([string(credentialsId: 'github', variable: 'GITHUB_TOKEN')]) {
          sh '''
            git config user.email "ishaanjohn21200@gmail.com"
            git config user.name "arther07git"
            sed -i "s/replaceImageTag/${BUILD_NUMBER}/g" spring-boot-app-manifests/deployment.yml
            git add spring-boot-app-manifests/deployment.yml
            git commit -m "Update deployment image to version ${BUILD_NUMBER}"
            git push https://${GITHUB_TOKEN}@github.com/${GIT_USER_NAME}/${GIT_REPO_NAME} HEAD:main
          '''
        }
      }
    }
  }
}
```

In your Jenkins pipeline, the following section defines the **agent** that Jenkins will use to execute the pipeline:

```groovy
agent {
    docker {
      image 'abhishekf5/maven-abhishek-docker-agent:v1'
      args '--user root -v /var/run/docker.sock:/var/run/docker.sock'
    }
  }
```

### **Breaking it Down:**
1. **`agent { docker { ... } }`**  
   - This tells Jenkins to use a **Docker container** as the agent instead of running the pipeline directly on the Jenkins host.

2. **`image 'abhishekf5/maven-abhishek-docker-agent:v1'`**  
   - Specifies that Jenkins should pull and run the Docker image `abhishekf5/maven-abhishek-docker-agent:v1`.
   - This image likely contains all dependencies required for building and testing the application (such as Maven, Java, and Docker CLI).

3. **`args '--user root -v /var/run/docker.sock:/var/run/docker.sock'`**  
   - `--user root`: Runs the container as the **root** user. This is required in some cases when the container needs elevated permissions.
   - `-v /var/run/docker.sock:/var/run/docker.sock`: Mounts the Docker daemon socket from the host into the container.  
     - This allows the Jenkins agent inside the container to communicate with the host’s Docker engine.  
     - This setup enables **Docker-in-Docker (DinD)**, meaning the pipeline can build and push Docker images.

### **Why Is This Used?**
- Running Jenkins agents inside a Docker container ensures a **clean, isolated, and reproducible environment** for builds.
- The mounted Docker socket allows the containerized Jenkins agent to **build and push Docker images** without needing a separate Docker installation inside the container.

### **Why Use Docker as an Agent in Jenkins?**
Instead of running the Jenkins pipeline directly on the host machine, you can use a **Docker container** as an agent. Here’s why this is beneficial:

---

### **Ensures a Clean, Isolated Build Environment**
- Every pipeline execution runs inside a **fresh container** based on the specified Docker image.
- Avoids dependency conflicts between different builds.
- Prevents build failures caused by host system changes.


### **Why Not Run Directly on the Jenkins Host?**
✅ Running directly on the host **works** but has downsides:
❌ Dependency conflicts between different builds.  
❌ Pollutes the Jenkins host with different tool installations.  
❌ Harder to maintain and upgrade environments.  
❌ Security risks if multiple jobs modify system files.  

---

### **When Should You Use Docker as an Agent?**
✅ When your builds need a **customized runtime** (specific versions of Java, Node.js, Python, etc.).  
✅ When you want to **run builds in isolated environments** to prevent conflicts.  
✅ When your pipeline involves **building and pushing Docker images**.  
✅ When you want **portability** and the same build environment everywhere.  


### **Pipeline Explanation**
| **Stage** | **Purpose** |
|------------|------------|
| Checkout | Fetches the latest code from GitHub. |
| Build & Test | Compiles the Java project using Maven and runs tests. |
| Build & Push Docker Image | Creates a Docker image and pushes it to Docker Hub. |
| Update Deployment File | Updates `deployment.yml` with the new image tag and pushes it to GitHub. |

---

## **Dockerfile (`Dockerfile`)**

```dockerfile
FROM adoptopenjdk/openjdk11:alpine-jre
ARG artifact=target/spring-boot-web.jar
WORKDIR /opt/app
COPY ${artifact} app.jar
ENTRYPOINT ["java","-jar","app.jar"]
```

### **Dockerfile Explanation**
- Uses `openjdk11:alpine-jre` as the base image.
- Copies the built JAR file into the container.
- Runs the JAR file using Java.

---

## **Kubernetes Deployment (`deployment.yml`)**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: spring-boot-app
  labels:
    app: spring-boot-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: spring-boot-app
  template:
    metadata:
      labels:
        app: spring-boot-app
    spec:
      containers:
      - name: spring-boot-app
        image: puneeth11/newultimate-cicd-pipeline:2
        ports:
        - containerPort: 8080
```

### **Deployment File Explanation**
- **Creates a deployment with 2 replicas**.
- **Uses the Docker image** (`puneeth11/newultimate-cicd-pipeline:2`).
- **Exposes port 8080** for the application.
- **Automatically updates the image tag** during deployment.

---

## **Kubernetes Service (`service.yml`)**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: spring-boot-app-service
spec:
  type: NodePort
  ports:
  - name: http
    port: 80
    targetPort: 8080
    protocol: TCP
  selector:
    app: spring-boot-app
```

### **Service File Explanation**
- Exposes the application using **NodePort**.
- Routes traffic from **port 80 → 8080** inside the container.
- Selects pods labeled `app: spring-boot-app`.

---

## **How Everything Works Together**
1. **Jenkins builds and tests the Spring Boot application**.
2. **Docker builds and pushes the image to Docker Hub**.
3. **Kubernetes pulls the new image and updates the deployment**.
4. **The application is exposed using a Kubernetes Service**.

---

## **Deployment Steps**
### **1. Run Jenkins Pipeline**
- Ensure Jenkins is running and has Docker permissions.
- Configure **GitHub and Docker credentials** in Jenkins.
- Trigger a build to execute the pipeline.

### **2. Apply Kubernetes Files**
```sh
kubectl apply -f deployment.yml
kubectl apply -f service.yml
```
### **3. Access the Application**
```sh
minikube service spring-boot-app-service --url
```
OR
```sh
kubectl get svc
```

---
## **Summary**
| **Component** | **Description** |
|------------|------------|
| Jenkins | Automates CI/CD pipeline execution. |
| Docker | Builds and pushes the application container. |
| Kubernetes | Deploys and manages the application. |
| NodePort Service | Exposes the application externally. |

This ensures a fully automated CI/CD pipeline for **continuous deployment**. 🚀

# SonarQube Setup

## **🔑 Generating and Configuring SonarQube Token**
1. Generate a **SonarQube token** from the SonarQube UI.
2. Add this token as a **SonarQube credential** in Jenkins.
3. Install the **SonarQube Scanner plugin** in Jenkins.

---

## **🔍 Why Is SonarQube Failing in Jenkins?**
Your SonarQube instance is **running and accessible at `http://localhost:9000`**, but Jenkins Cannot reach it.

---

## **✅ Solution: Use the Correct SonarQube URL**
Modify the **SonarQube stage** in your `Jenkinsfile` to use the **host machine's IP (private IP)** instead of `localhost`.

### **🔹 Step 1: Find Your Host Machine's IP**
Run the following command on the host where Jenkins is running:
```bash
ip a | grep inet
```
Example output:
```
inet 192.XXX.X.1X0/24 brd 192.168.1.255 scope global eth0
```
🔹 **Use the extracted IP (e.g., `192.1XX.1.XXX`) in your Jenkins configuration.**

---

### **🔹 Step 2: Update the `Jenkinsfile`**
Modify the SonarQube stage in your `Jenkinsfile`:
```groovy
stage('Static Code Analysis') {
    environment {
        SONAR_URL = "http://192.XXX.1.100:9000" // Use your host machine's IP
    }
    steps {
        withCredentials([string(credentialsId: 'sonarqube', variable: 'SONAR_AUTH_TOKEN')]) {
            sh 'cd spring-boot-app && mvn sonar:sonar -Dsonar.login=$SONAR_AUTH_TOKEN -Dsonar.host.url=${SONAR_URL}'
        }
    }
}
```
🔹 **Replace `192.168.1.100` with your actual host machine IP.**

---

### **🔹 Step 3: Restart Jenkins and Retry the Pipeline**
Restart Jenkins to apply changes:
```bash
sudo systemctl restart jenkins
```
Then, rerun your pipeline.


🚀 **Your SonarQube setup should now work smoothly with Jenkins!**



# **Explanation of Jenkins Stages: "Build and Push Docker Image" & "Update Deployment File"**  

These two stages handle **containerization, pushing the image to Docker Hub, and updating the Kubernetes deployment file** in GitHub.  

---

## **1️⃣ Stage: "Build and Push Docker Image"**  
This stage is responsible for **building the Docker image** from the JAR file and pushing it to Docker Hub.

### **Environment Variables**  
```groovy
environment {
    DOCKER_IMAGE = "puneeth11/newultimate-cicd-pipeline:${BUILD_NUMBER}"
    REGISTRY_CREDENTIALS = credentials('docker-cred')
}
```
- **`DOCKER_IMAGE`** – Defines the image name and tag (`BUILD_NUMBER` ensures each build gets a unique tag).  
  - Example: `puneeth11/newultimate-cicd-pipeline:10` (if `BUILD_NUMBER = 10`)  
- **`REGISTRY_CREDENTIALS`** – Retrieves Docker Hub credentials from Jenkins using `credentials('docker-cred')`.  

### **Steps (Building & Pushing the Image)**  
```groovy
script {
    sh 'cd spring-boot-app && docker build -t ${DOCKER_IMAGE} .'
    def dockerImage = docker.image("${DOCKER_IMAGE}")
    docker.withRegistry('https://index.docker.io/v1/', "docker-cred") {
        dockerImage.push()
    }
}
```

#### **Step 1: Build Docker Image**
```sh
cd spring-boot-app && docker build -t ${DOCKER_IMAGE} .
```
- Moves into the `spring-boot-app` directory.  
- Builds a Docker image using the `Dockerfile`.  
- The image is tagged as `"puneeth11/newultimate-cicd-pipeline:${BUILD_NUMBER}"`.  

#### **Step 2: Push Docker Image to Docker Hub**
```groovy
def dockerImage = docker.image("${DOCKER_IMAGE}")
docker.withRegistry('https://index.docker.io/v1/', "docker-cred") {
    dockerImage.push()
}
```
- Uses Jenkins' `docker.image()` to reference the newly built image.  
- `docker.withRegistry()` authenticates with Docker Hub using `docker-cred`.  
- Pushes the image to the Docker Hub repository **"puneeth11/newultimate-cicd-pipeline"** with the **`${BUILD_NUMBER}`** tag.  

**Result:**  
The image is now stored in Docker Hub and can be pulled by Kubernetes.

---

## **2️⃣ Stage: "Update Deployment File"**  
This stage **updates the Kubernetes deployment file (`deployment.yml`)** to use the newly built Docker image.  

### **Environment Variables**  
```groovy
environment {
    GIT_REPO_NAME = "Java-Applcation-CI-CD"
    GIT_USER_NAME = "arther07git"
}
```
- **`GIT_REPO_NAME`** – Name of the GitHub repository.  
- **`GIT_USER_NAME`** – GitHub username.  

### **Steps (Updating Kubernetes YAML File in GitHub)**  
```groovy
withCredentials([string(credentialsId: 'github', variable: 'GITHUB_TOKEN')]) {
    sh '''
        git config user.email "ishaanjohn21200@gmail.com"
        git config user.name "arther07git"
        BUILD_NUMBER=${BUILD_NUMBER}
        sed -i "s/replaceImageTag/${BUILD_NUMBER}/g" spring-boot-app-manifests/deployment.yml
        git add spring-boot-app-manifests/deployment.yml
        git commit -m "Update deployment image to version ${BUILD_NUMBER}"
        git push https://${GITHUB_TOKEN}@github.com/${GIT_USER_NAME}/${GIT_REPO_NAME} HEAD:main
    '''
}
```

#### **Step 1: Authenticate with GitHub**  
```groovy
withCredentials([string(credentialsId: 'github', variable: 'GITHUB_TOKEN')])
```
- Fetches **GitHub credentials** stored in Jenkins (`github` ID).  
- Stores the GitHub token in the `GITHUB_TOKEN` variable.  

#### **Step 2: Configure Git User Details**  
```sh
git config user.email "ishaanjohn21200@gmail.com"
git config user.name "arther07git"
```
- Configures Git **user details** (used for committing changes).  

#### **Step 3: Update Deployment File with New Image**  
```sh
sed -i "s/replaceImageTag/${BUILD_NUMBER}/g" spring-boot-app-manifests/deployment.yml
```
- **Finds and replaces** `replaceImageTag` in `deployment.yml` with `${BUILD_NUMBER}`.  
- Example: If `BUILD_NUMBER=10`, the YAML file updates to:  
  ```yaml
  image: puneeth11/newultimate-cicd-pipeline:10
  ```
  This ensures Kubernetes pulls the latest image.


#### **Step 4: Commit & Push Changes to GitHub**
```sh
git add spring-boot-app-manifests/deployment.yml
git commit -m "Update deployment image to version ${BUILD_NUMBER}"
git push https://${GITHUB_TOKEN}@github.com/${GIT_USER_NAME}/${GIT_REPO_NAME} HEAD:main
```
- **Adds the updated file** to Git.  
- **Commits the change** with a message: `"Update deployment image to version ${BUILD_NUMBER}"`.  
- **Pushes the changes** to the GitHub repository.  

---

### **Final Result**
✔ Docker image is built & pushed to Docker Hub.  
✔ `deployment.yml` is updated with the latest image tag.  
✔ GitHub repository is updated.  
✔ Kubernetes will now use the latest image when redeploying the application. 

# **Explanation of the `sed` Command:**
```sh
sed -i "s/replaceImageTag/${BUILD_NUMBER}/g" spring-boot-app-manifests/deployment.yml
```
This command **modifies the `deployment.yml` file** by replacing `replaceImageTag` with the actual **`${BUILD_NUMBER}`** value.

---

### **Breaking It Down:**
1. **`sed`** → Stream Editor, used for text manipulation.  
2. **`-i`** → Edits the file **in place** (modifies the file directly).  
3. **`s/replaceImageTag/${BUILD_NUMBER}/g`**  
   - **`s/`** → Substitutes text.  
   - **`replaceImageTag`** → The placeholder text to be replaced in the `deployment.yml` file.  
   - **`${BUILD_NUMBER}`** → The Jenkins build number (e.g., `10`, `11`, `12` …).  
   - **`/g`** → **Global replacement**, ensuring all occurrences of `replaceImageTag` are replaced.  
4. **`spring-boot-app-manifests/deployment.yml`** → The file being modified.

---

### **Example:**
#### **Before Modification (`deployment.yml`):**
```yaml
containers:
  - name: spring-boot-app
    image: puneeth11/newultimate-cicd-pipeline:replaceImageTag
    ports:
      - containerPort: 8080
```

#### **Jenkins Runs: (`BUILD_NUMBER=10`)**
```sh
sed -i "s/replaceImageTag/10/g" spring-boot-app-manifests/deployment.yml
```

#### **After Modification (`deployment.yml`):**
```yaml
containers:
  - name: spring-boot-app
    image: puneeth11/newultimate-cicd-pipeline:10
    ports:
      - containerPort: 8080
```

---

### **Why is this Important?**
✔ **Ensures Kubernetes always pulls the latest image** when a new deployment happens.  
✔ **Automates the process** of updating the deployment without manual edits.  
✔ Works **dynamically with Jenkins**, as each build gets a unique number (`BUILD_NUMBER`).  

## But
After the first replacement, `replaceImageTag` no longer exists in `deployment.yml`, so running the same `sed` command won't work for the next build (BUILD_NUMBER=2).  

### Solution: Modify the `sed` Command  
Instead of searching for `replaceImageTag`, modify the script to replace **any existing tag** in the image section.  

#### **Updated `sed` Command:**
```sh
sed -i "s|image: puneeth11/newultimate-cicd-pipeline:[^ ]*|image: puneeth11/newultimate-cicd-pipeline:${BUILD_NUMBER}|" spring-boot-app-manifests/deployment.yml        
```

### **How It Works**
1. **Finds the existing image tag:**  
   - It looks for `image: puneeth11/newultimate-cicd-pipeline:<anything_here>`  
   - `[^ ]*` matches any non-space characters (i.e., the previous version like `1`, `2`, `10`, etc.)  
   
2. **Replaces it with the new `BUILD_NUMBER`.**  

### **Example Workflow**
#### **First Run (BUILD_NUMBER=1)**
Before:
```yaml
containers:
  - name: spring-boot-app
    image: puneeth11/newultimate-cicd-pipeline:replaceImageTag
    ports:
      - containerPort: 8080
```
After running:
```yaml
containers:
  - name: spring-boot-app
    image: puneeth11/newultimate-cicd-pipeline:1
    ports:
      - containerPort: 8080
```

#### **Second Run (BUILD_NUMBER=2)**
Before:
```yaml
containers:
  - name: spring-boot-app
    image: puneeth11/newultimate-cicd-pipeline:1
    ports:
      - containerPort: 8080
```
After running:
```yaml
containers:
  - name: spring-boot-app
    image: puneeth11/newultimate-cicd-pipeline:2
    ports:
      - containerPort: 8080
```

### **Why This Works?**
- The previous version (e.g., `image: ...:1`) is updated dynamically.
- This ensures the `deployment.yml` is always updated to the latest build version.  

Let me know if you need any modifications! 🚀


# ArgoCD Setup for Continuous Delivery in Jenkins Pipeline

## Prerequisites
- A running **Minikube** cluster
- `kubectl` installed and configured
- `helm` installed
- `minikube status` should show that the cluster is running

---

## 1. Install ArgoCD Using OperatorHub.io Documentation
ArgoCD can be installed using the OperatorHub.io method or by applying a YAML file.

### Option 1: Install via OperatorHub.io
Follow the instructions from [OperatorHub.io](https://operatorhub.io/operator/argocd).

### Option 2: Install via YAML Manifest
Create a YAML file (`argocd-basic.yml`) and copy the following contents:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ArgoCD
metadata:
  name: example-argocd
  labels:
    example: basic
spec:
  server:
    service:
      type: NodePort
```

Apply the YAML file:
```sh
kubectl apply -f argocd-basic.yml
```

Verify ArgoCD installation:
```sh
kubectl get pods -n argocd
```

---

## 2. Change ArgoCD Service Type
By default, ArgoCD is set to `ClusterIP`. To access it externally, change it to `NodePort`.

Edit the service:
```sh
kubectl edit svc example-argocd-server -n argocd
```
Change:
```yaml
spec:
  type: NodePort  # Change from ClusterIP to NodePort
```
Save and exit.

Expose ArgoCD:
```sh
minikube service example-argocd-server -n argocd
```
This will return a URL. Open it in your browser to access ArgoCD.

---

## 3. Login to ArgoCD
Retrieve the admin password:
```sh
kubectl get secret example-argocd-cluster -n argocd -o jsonpath="{.data.admin\.password}" | base64 -d
```

- **Username:** `admin`
- **Password:** `<decoded password>`

Login to ArgoCD:
```sh
argocd login <ARGOCD_URL>:<NODEPORT> --username admin --password <decoded password>
```

---

## 4. Create a New Project in ArgoCD
1. Open the ArgoCD UI in a browser using the provided Minikube URL.
2. Click **New Project**.
3. Give it a name and assign a namespace (`default` if unsure).
4. Select the **source code repository** (GitHub repository containing Kubernetes manifests).
5. Select the **destination cluster** (Minikube cluster suggestion).
6. Click **Create**.

Verify the project:
```sh
kubectl get deploy -n argocd
kubectl get pods -n argocd
```

---

## 5. Deploy and Access the Application
ArgoCD will now automatically sync and deploy applications.

To check deployment:
```sh
kubectl get deploy -n default
kubectl get pods -n default
```

To access the Spring Boot application:
```sh
minikube service spring-boot-app-server-service -n default
```
This will return a URL with a **NodePort**, which you can open in your browser to access the deployed application.

---

## 🎉 Done!
You have successfully set up **ArgoCD for Continuous Delivery** in your Minikube environment.. 🚀


# NOTE
# Jenkins Pipeline with Docker and Maven

## Overview
This document explains the functionality of the provided Jenkinsfile, including the use of Docker and Maven in a Jenkins pipeline. Additionally, it covers alternative setups if Docker is not used as an agent and how to configure Jenkins to run Maven and Docker commands directly.

---

## Jenkinsfile Breakdown

### Docker Agent Configuration
```groovy
agent {
  docker {
    image 'abhishekf5/maven-abhishek-docker-agent:v1'
    args '--user root -v /var/run/docker.sock:/var/run/docker.sock'
  }
}
```

### Explanation:
1. **`image 'abhishekf5/maven-abhishek-docker-agent:v1'`**  
   - Specifies that the pipeline will run inside a **Docker container**.
   - The container uses an image that includes **Maven** and **Docker CLI**, reducing the need for installing dependencies on the Jenkins server.

2. **`args '--user root -v /var/run/docker.sock:/var/run/docker.sock'`**  
   - **`--user root`** → Runs the container as the **root** user to avoid permission issues.
   - **`-v /var/run/docker.sock:/var/run/docker.sock`** → Mounts the Docker socket from the host, allowing the container to access the host's Docker daemon.
  
     The argument:  
```groovy
args '--user root -v /var/run/docker.sock:/var/run/docker.sock'
```
is used to modify the behavior of the **Jenkins Docker agent container**. Here's what each part does:

---

### **1️⃣ `--user root`**
- Runs the container as the **root user**.
- By default, containers may run as a non-root user, which can cause permission issues when trying to access system resources like Docker or files.
- Running as **root** ensures the container has the necessary permissions to execute all commands (like Maven builds, Docker builds, and pushing images).

---

### **2️⃣ `-v /var/run/docker.sock:/var/run/docker.sock`**
- **Mounts the host machine’s Docker socket inside the container.**
- The `/var/run/docker.sock` file is a **UNIX socket** used to communicate with the Docker daemon on the host machine.
- By mounting it, the container gets access to the **host's Docker engine**, enabling it to execute Docker commands **as if they were run on the host**.

---

### **Why is this needed?**
✅ **Without this argument:**
- The container will not have access to the host's Docker engine.
- Any `docker build` or `docker push` command inside the container **will fail** because there is no Docker daemon running inside the container itself.

✅ **With this argument:**
- The container acts like a "remote client" to the host’s Docker daemon.
- Any `docker` command inside the container (like `docker build` or `docker push`) is executed on the **host's Docker engine**, avoiding the need to install Docker inside the container.

---

### **Summary**
| Argument | Purpose |
|----------|---------|
| `--user root` | Runs the container as root to avoid permission issues. |
| `-v /var/run/docker.sock:/var/run/docker.sock` | Allows the container to communicate with the host's Docker daemon, enabling it to build and push images. |

Would you like me to update this explanation in the README.md file? 🚀

### Why is this needed?
- Since the Jenkins pipeline runs inside a **Docker container**, it does not have a Docker daemon by default.
- By mounting `/var/run/docker.sock`, it enables the container to **directly communicate with the host’s Docker daemon**, allowing it to build and push images.

---

## Alternative Setup (Without Docker Agent)
If you **do not use** the pre-configured Docker image (`abhishekf5/maven-abhishek-docker-agent:v1`), you need to set up Maven and Docker manually on the Jenkins server.

### 1️⃣ Install Maven on Jenkins Server
#### **On Ubuntu/Debian:**
```bash
sudo apt update
sudo apt install maven -y
mvn -version  # Verify installation
```
#### **On CentOS/RHEL:**
```bash
sudo yum install maven -y
mvn -version  # Verify installation
```

### 2️⃣ Install the Maven Integration Plugin in Jenkins
- Go to **Jenkins Dashboard** → **Manage Jenkins** → **Manage Plugins**.
- Search for **Maven Integration Plugin** and install it.

### 3️⃣ Configure Maven in Jenkins Tools Section
- Go to **Manage Jenkins** → **Global Tool Configuration**.
- Under **Maven**, click **Add Maven**.
- Set:
  - **Name** → `Maven-3.8.8`
  - **Installation Directory** (or let Jenkins install automatically).
- Save the settings.

### 4️⃣ Update Jenkinsfile (Using Installed Maven)
#### **BEFORE (Using Docker Agent)**
```groovy
agent {
  docker {
    image 'abhishekf5/maven-abhishek-docker-agent:v1'
    args '--user root -v /var/run/docker.sock:/var/run/docker.sock'
  }
}
```
#### **AFTER (Using Installed Maven on Jenkins Server)**
```groovy
agent any  // Run on any available Jenkins node
tools {
  maven 'Maven-3.8.8'  // Use the configured Maven version
}
```

Now, Maven commands will execute successfully in Jenkins pipeline.

---

## Running Docker Commands in Jenkins Pipeline
If you remove the Docker agent, you must install Docker on the Jenkins server and ensure Jenkins has permission to run Docker commands.

### 1️⃣ Install Docker on Jenkins Server
#### **On Ubuntu/Debian:**
```bash
sudo apt update
sudo apt install docker.io -y
sudo systemctl enable docker
sudo systemctl start docker
docker --version  # Verify installation
```
#### **On CentOS/RHEL:**
```bash
sudo yum install docker -y
sudo systemctl enable docker
sudo systemctl start docker
docker --version  # Verify installation
```

### 2️⃣ Add Jenkins User to Docker Group
By default, the `jenkins` user does not have permission to run Docker commands. Add it to the `docker` group:
```bash
sudo usermod -aG docker jenkins
sudo systemctl restart jenkins
```
✅ Now, Jenkins can execute Docker commands **without needing `sudo`**.

### 3️⃣ Update Jenkinsfile (Without Docker Agent)
#### **BEFORE (Using Docker Agent)**
```groovy
agent {
  docker {
    image 'abhishekf5/maven-abhishek-docker-agent:v1'
    args '--user root -v /var/run/docker.sock:/var/run/docker.sock'
  }
}
```
#### **AFTER (Using Installed Docker on Jenkins Server)**
```groovy
pipeline {
  agent any  // Run on any available Jenkins node
  tools {
    maven 'Maven-3.8.8'  // Use the configured Maven
  }
  environment {
    DOCKER_IMAGE = "puneeth11/newultimate-cicd-pipeline:${BUILD_NUMBER}"
  }
  stages {
    stage('Build Docker Image') {
      steps {
        sh 'docker build -t ${DOCKER_IMAGE} .'
      }
    }
    stage('Push to Docker Hub') {
      steps {
        withCredentials([string(credentialsId: 'docker-cred', variable: 'DOCKER_PASSWORD')]) {
          sh '''
            echo "$DOCKER_PASSWORD" | docker login -u "your-dockerhub-username" --password-stdin
            docker push ${DOCKER_IMAGE}
          '''
        }
      }
    }
  }
}
```

---

## ✅ **Steps to Configure Maven in Jenkins Dashboard**
1. **Go to Jenkins Dashboard (`http://your-jenkins-server:8080`).**
2. Click on **Manage Jenkins** → **Global Tool Configuration**.
3. Scroll down to the **Maven** section.
4. Click **Add Maven** and configure:
   - **Name**: `Maven-3.8.8` (or any version you installed).
   - **Installation Method**: Let Jenkins install it automatically or specify a manually installed Maven path.
5. **Save** the configuration.

### 🔥 **Final Setup Checklist**
| Step | Required? | Location |
|------|----------|-----------|
| Install Maven | ✅ Yes (if not using Docker agent) | Server CLI (`sudo apt install maven -y`) |
| Configure Maven in Jenkins | ✅ Yes | `Manage Jenkins` → `Global Tool Configuration` |
| Install Docker | ✅ Yes (if not using Docker agent) | Server CLI (`sudo apt install docker.io -y`) |
| Add Jenkins to Docker group | ✅ Yes | `sudo usermod -aG docker jenkins` |
| Update `PATH` variable (if needed) | ⚠️ Maybe | `Manage Jenkins` → `Global Tool Configuration` |

✅ **Once this is done, your Jenkins pipeline will run both Maven and Docker commands successfully!** 🚀


# NOTE 

### **What Does `agent any` Mean in Jenkins Pipeline?**
```groovy
agent any
```
This line in a **Jenkins Declarative Pipeline** means:

1️⃣ The pipeline **can run on any available Jenkins agent (node)**, including:
   - The **Jenkins master node** (if allowed).
   - Any **connected worker node** (if Jenkins is running in a distributed setup).

2️⃣ Jenkins will **automatically select an available agent** to execute the pipeline.

---

### **Comparison: `agent any` vs. `agent { docker { ... } }`**
| Configuration | What Happens? |
|--------------|--------------|
| `agent any` | Runs on any available Jenkins agent (master or worker nodes). |
| `agent { label 'docker' }` | Runs only on a specific agent with the label `docker`. |
| `agent { docker { image 'maven:3.8.8' } }` | Runs inside a **Docker container** with the `maven:3.8.8` image. |

---

### **When to Use `agent any`?**
- When you **don't need a specific environment** (like a Docker container or a specific agent label).
- When Jenkins can run the job on any available node in a **multi-node setup**.

✅ **If you remove the Docker agent and use `agent any`, ensure that Maven and Docker are installed on the Jenkins server!**  

# NOTE

### **Where Does `agent { docker { image 'maven:3.8.8' } }` Run?**  
When you specify this in a Jenkins pipeline:  
```groovy
agent { 
    docker { 
        image 'maven:3.8.8' 
    } 
}
```
this means **Jenkins will start a new container** with the `maven:3.8.8` image and run all pipeline steps inside it. But where does this container run?

---

### **1️⃣ If Jenkins is running as a standalone server (no worker nodes)**
✅ **Scenario:**  
- Jenkins is installed directly on a **bare-metal or VM** (e.g., on an EC2 instance or an Ubuntu server).  
- There are **no separate worker nodes**.  

🔹 **What happens?**  
- The **Jenkins master itself** will create a **Docker container** using the `maven:3.8.8` image.  
- The entire pipeline runs inside this container.  
- Once the pipeline execution is complete, the container is **removed** (unless `--rm=false` is set).  

**Example Execution Flow:**
1. Jenkins **pulls the Docker image** `maven:3.8.8` (if not already present).
2. Jenkins **creates a Docker container** with this image.
3. Jenkins **executes the pipeline steps** inside the container.
4. After execution, the container **stops and is removed**.

✅ Prerequisites for Running Jenkins with a Docker Agent
Since the pipeline executes inside a Docker container, Jenkins needs access to Docker to create and manage containers.

1️⃣ Install Docker on Jenkins Server
If Docker is not installed, the pipeline will fail. 

---

### **2️⃣ If Jenkins is running with worker nodes (distributed setup)**
✅ **Scenario:**  
- Jenkins is running in a **master-agent (worker) setup**.  
- The agent (worker node) is responsible for running jobs.  

🔹 **What happens?**  
- The Jenkins **worker node** (where the job runs) creates a **Docker container** using `maven:3.8.8`.  
- The pipeline executes inside this container, isolated from the Jenkins agent.  
- The container **stops and is removed** once the job completes.

**Example Execution Flow:**
1. Jenkins master **assigns the job** to a worker node.
2. The worker node **pulls the Docker image** `maven:3.8.8` (if not already present).
3. The worker node **creates a container** with the image.
4. Jenkins **executes pipeline steps** inside this container.
5. After execution, the container **stops and is removed**.

---

### **Summary: Where Does It Run?**
| **Jenkins Setup** | **Where the Docker Container Runs?** |
|------------------|----------------------------------|
| **Standalone Jenkins Server (no agents)** | Runs inside the **Jenkins master server**. |
| **Master-Agent Jenkins Setup** | Runs inside the **assigned worker node** (not the master). |

### **Key Takeaways**
✅ The pipeline runs **inside a container**, **not directly on the host system**.  
✅ The container is **temporary** and gets **deleted after the job completes**.  
✅ If Docker is **not installed** on the Jenkins master/agent, the pipeline **will fail** because it cannot create a container.  

---



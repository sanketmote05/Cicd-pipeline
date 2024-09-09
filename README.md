### **Step-by-Step Guide to Setting Up the CI/CD Project**

---

### **Step 1: Create a GitHub Repository**
1. **Go to GitHub** and create a new repository named `ci-cd-pipeline-demo`.
2. Clone the repository to your local machine:
   ```bash
   git clone https://github.com/yourusername/ci-cd-pipeline-demo.git
   cd ci-cd-pipeline-demo
   ```

---

### **Step 2: Set Up Your Java Maven Project**
1. **Create the directory structure**:
   ```bash
   mkdir -p src/main/java src/test/java k8s
   ```

2. **Add the Java Application** (`App.java`):
   - Create `src/main/java/App.java`:
     ```java
     public class App {
         public static void main(String[] args) {
             System.out.println("Hello, World! Simple Java Application for CI/CD Pipeline.");
         }
     }
     ```

3. **Add a Basic Test Class** (`AppTest.java`):
   - Create `src/test/java/AppTest.java`:
     ```java
     import static org.junit.Assert.assertTrue;
     import org.junit.Test;

     public class AppTest {
         @Test
         public void shouldAnswerWithTrue() {
             assertTrue(true);
         }
     }
     ```

4. **Create a `pom.xml`** for the Maven project:
   - Add this file in the root directory:
     ```xml
     <project xmlns="http://maven.apache.org/POM/4.0.0"
              xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
              xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
         <modelVersion>4.0.0</modelVersion>
         <groupId>com.example</groupId>
         <artifactId>ci-cd-pipeline-demo</artifactId>
         <version>1.0-SNAPSHOT</version>

         <properties>
             <maven.compiler.source>11</maven.compiler.source>
             <maven.compiler.target>11</maven.compiler.target>
         </properties>

         <dependencies>
             <dependency>
                 <groupId>junit</groupId>
                 <artifactId>junit</artifactId>
                 <version>4.12</version>
                 <scope>test</scope>
             </dependency>
         </dependencies>

         <build>
             <plugins>
                 <plugin>
                     <groupId>org.apache.maven.plugins</groupId>
                     <artifactId>maven-compiler-plugin</artifactId>
                     <version>3.8.1</version>
                     <configuration>
                         <source>11</source>
                         <target>11</target>
                     </configuration>
                 </plugin>
             </plugins>
         </build>
     </project>
     ```

5. **Push these files to GitHub**:
   ```bash
   git add .
   git commit -m "Add initial Maven project with Java app and tests"
   git push origin main
   ```

---

### **Step 3: Create a Dockerfile**
1. **Create the `Dockerfile`** in the root of the project:
   ```Dockerfile
   FROM openjdk:11-jdk-slim
   COPY target/myapp.jar /app/myapp.jar
   ENTRYPOINT ["java", "-jar", "/app/myapp.jar"]
   ```

2. **Push the `Dockerfile`**:
   ```bash
   git add Dockerfile
   git commit -m "Add Dockerfile for Java app"
   git push origin main
   ```

---

### **Step 4: Set Up Kubernetes YAML Files**
1. **Create the Deployment file** (`deployment.yml`):
   - Add `k8s/deployment.yml`:
     ```yaml
     apiVersion: apps/v1
     kind: Deployment
     metadata:
       name: myapp
     spec:
       replicas: 1
       selector:
         matchLabels:
           app: myapp
       template:
         metadata:
           labels:
             app: myapp
         spec:
           containers:
           - name: myapp
             image: yourusername/myapp:latest
             ports:
             - containerPort: 8080
     ```

2. **Create the Service file** (`service.yml`):
   - Add `k8s/service.yml`:
     ```yaml
     apiVersion: v1
     kind: Service
     metadata:
       name: myapp-service
     spec:
       selector:
         app: myapp
       ports:
         - protocol: TCP
           port: 80
           targetPort: 8080
       type: ClusterIP
     ```

3. **Create the Ingress file** (`ingress.yml`):
   - Add `k8s/ingress.yml`:
     ```yaml
     apiVersion: networking.k8s.io/v1
     kind: Ingress
     metadata:
       name: myapp-ingress
       annotations:
         nginx.ingress.kubernetes.io/rewrite-target: /
     spec:
       rules:
         - host: myapp.local
           http:
             paths:
               - path: /
                 pathType: Prefix
                 backend:
                   service:
                     name: myapp-service
                     port:
                       number: 80
     ```

4. **Push the YAML files to GitHub**:
   ```bash
   git add k8s/
   git commit -m "Add Kubernetes deployment, service, and ingress files"
   git push origin main
   ```

---

### **Step 5: Create a Jenkins Pipeline (`Jenkinsfile`)**
1. **Create `Jenkinsfile`** in the root of the project:
   ```groovy
   pipeline {
       agent any
       
       tools {
           maven 'Maven'
       }
       
       stages {
           stage('Checkout') {
               steps {
                   git branch: 'main', url: 'https://github.com/yourusername/ci-cd-pipeline-demo.git'
               }
           }
           
           stage('Build') {
               steps {
                   sh 'mvn clean install'
               }
           }
           
           stage('SonarQube Analysis') {
               steps {
                   withSonarQubeEnv('SonarQube') {
                       sh 'mvn sonar:sonar'
                   }
               }
           }
           
           stage('Docker Build') {
               steps {
                   sh 'docker build -t yourusername/myapp:latest .'
               }
           }
           
           stage('Docker Push') {
               steps {
                   withCredentials([string(credentialsId: 'dockerhub', variable: 'DOCKERHUB_PASSWORD')]) {
                       sh "echo $DOCKERHUB_PASSWORD | docker login -u yourusername --password-stdin"
                       sh 'docker push yourusername/myapp:latest'
                   }
               }
           }
           
           stage('Update Deployment YAML') {
               steps {
                   sh 'sed -i "s|image:.*|image: yourusername/myapp:latest|" k8s/deployment.yml'
                   git credentialsId: 'github-creds', url: 'https://github.com/yourusername/ci-cd-pipeline-demo.git'
                   sh 'git add k8s/deployment.yml'
                   sh 'git commit -m "Update deployment.yml with new Docker image"'
                   sh 'git push'
               }
           }
       }
   }
   ```

2. **Push the `Jenkinsfile`**:
   ```bash
   git add Jenkinsfile
   git commit -m "Add Jenkinsfile for CI/CD pipeline"
   git push origin main
   ```

---

### **Updated Step-by-Step for ArgoCD Deployment**


---

### **Step 6: Set Up ArgoCD for Continuous Deployment**

1. **Install ArgoCD**:
   - If you haven't installed ArgoCD in your Kubernetes cluster yet, follow these steps:
     ```bash
     kubectl create namespace argocd
     kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
     ```

2. **Access ArgoCD UI**:
   - Once ArgoCD is installed, expose the ArgoCD server to access the web UI:
     ```bash
     kubectl port-forward svc/argocd-server -n argocd 8080:443
     ```
   - Access the ArgoCD UI by navigating to `https://localhost:8080` in your browser.

3. **Log in to ArgoCD**:
   - The initial admin password can be retrieved with the following command:
     ```bash
     kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 --decode; echo
     ```

4. **Create an ArgoCD Application**:
   - Go to the ArgoCD UI and create a new application:
     - **Application Name**: `myapp`
     - **Project**: `default`
     - **Sync Policy**: Automatic
     - **Repository URL**: `https://github.com/yourusername/ci-cd-pipeline-demo.git`
     - **Path**: `k8s/` (this is where the deployment, service, and ingress files are stored)
     - **Cluster URL**: `https://kubernetes.default.svc`
     - **Namespace**: `default` (or the namespace you are using)

5. **Configure Auto-Sync**:
   - Set the application’s sync policy to **Auto-Sync** so that every time you push changes to the `k8s/` folder (e.g., when Jenkins updates the `deployment.yml` with the new Docker image), ArgoCD will automatically deploy the latest changes to the Kubernetes cluster.

---

### **Step 7: Trigger the Deployment via Jenkins**

1. **Jenkins Pipeline Update**:
   Since ArgoCD will handle the deployment automatically based on the changes in the Git repository, remove the `kubectl` commands and simply let Jenkins update the `deployment.yml` file with the new Docker image.

   Updated **Jenkinsfile**:
   ```groovy
   pipeline {
       agent any
       
       tools {
           maven 'Maven'
       }
       
       stages {
           stage('Checkout') {
               steps {
                   git branch: 'main', url: 'https://github.com/yourusername/ci-cd-pipeline-demo.git'
               }
           }
           
           stage('Build') {
               steps {
                   sh 'mvn clean install'
               }
           }
           
           stage('SonarQube Analysis') {
               steps {
                   withSonarQubeEnv('SonarQube') {
                       sh 'mvn sonar:sonar'
                   }
               }
           }
           
           stage('Docker Build') {
               steps {
                   sh 'docker build -t yourusername/myapp:latest .'
               }
           }
           
           stage('Docker Push') {
               steps {
                   withCredentials([string(credentialsId: 'dockerhub', variable: 'DOCKERHUB_PASSWORD')]) {
                       sh "echo $DOCKERHUB_PASSWORD | docker login -u yourusername --password-stdin"
                       sh 'docker push yourusername/myapp:latest'
                   }
               }
           }
           
           stage('Update Deployment YAML') {
               steps {
                   sh 'sed -i "s|image:.*|image: yourusername/myapp:latest|" k8s/deployment.yml'
                   git credentialsId: 'github-creds', url: 'https://github.com/yourusername/ci-cd-pipeline-demo.git'
                   sh 'git add k8s/deployment.yml'
                   sh 'git commit -m "Update deployment.yml with new Docker image"'
                   sh 'git push'
               }
           }
       }
   }
   ```

2. **How It Works**:
   - The pipeline will:
     - Build and test the Java application.
     - Run SonarQube analysis.
     - Build and push the Docker image to Docker Hub.
     - Update the `deployment.yml` file in GitHub with the new Docker image tag.
   - **ArgoCD** will detect this change in the GitHub repository and automatically sync the new `deployment.yml` file to deploy the new version of your application to Kubernetes.

---

### **Summary of the CI/CD Flow with ArgoCD**:
1. **Code Push**: Developers push code to the GitHub repository.
2. **Jenkins Pipeline**: Jenkins builds the code, runs tests, performs code analysis (SonarQube), builds the Docker image, and pushes it to Docker Hub. Then, it updates the `deployment.yml` in GitHub with the new image tag.
3. **ArgoCD Deployment**: ArgoCD monitors the repository for changes. When the `deployment.yml` is updated, ArgoCD automatically pulls the new configuration and deploys the updated Docker image to the Kubernetes cluster.

This setup ensures a fully automated CI/CD pipeline with continuous integration handled by Jenkins and continuous deployment handled by ArgoCD.

Step-by-step guide to creating a CI/CD project using Git, Jenkins, SonarQube, Maven, Docker, Docker Registry, and ArgoCD. This will walk you through each stage, from setting up your environment to automating the deployment.

### **Prerequisites:**
- Jenkins, SonarQube, and Docker installed on your local machine or server.
- DockerHub or a private Docker registry.
- ArgoCD setup on a Kubernetes cluster.
- GitHub account.

---

### **Step 1: Set Up Your GitHub Repository**
1. **Create a GitHub Repository**: Create a new GitHub repository called `ci-cd-pipeline-demo`.

2. **Add a Maven Project**:
   - Create a simple Maven project or use an existing one.
   - Here's a basic structure:
     ```bash
     ci-cd-pipeline-demo/
     ├── src/
     │   ├── main/
     │   │   └── java/
     │   │       └── App.java
     │   └── test/
     │       └── java/
     │           └── AppTest.java
     ├── pom.xml
     └── Dockerfile
     ```
   - Ensure `pom.xml` is configured to compile your project and run unit tests.

3. **Create a `Dockerfile`**:
   - Add a `Dockerfile` to containerize your Maven project.
     ```Dockerfile
     FROM openjdk:11-jdk-slim
     COPY target/myapp.jar /app/myapp.jar
     ENTRYPOINT ["java", "-jar", "/app/myapp.jar"]
     ```

4. **Push to GitHub**:
   - Commit and push the code to the GitHub repository.

---

### **Step 2: Set Up Jenkins Pipeline**
1. **Install Jenkins Plugins**:
   - Make sure you have the following plugins installed:
     - Git Plugin
     - Pipeline Plugin
     - Docker Pipeline Plugin
     - SonarQube Plugin
     - Maven Integration Plugin

2. **Create a New Jenkins Pipeline**:
   - Go to Jenkins Dashboard > New Item > Pipeline.
   - Name the job `ci-cd-pipeline`.
   
3. **Configure the Pipeline Script**:
   - Go to the “Pipeline” section and select "Pipeline script from SCM".
   - Choose `Git` and provide the GitHub repository URL for your Maven project.
   - In your repository, create a `Jenkinsfile`:
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
                     script {
                         sh 'mvn clean install'
                     }
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
                     script {
                         sh 'docker build -t yourusername/myapp:latest .'
                     }
                 }
             }
             
             stage('Docker Push') {
                 steps {
                     script {
                         withCredentials([string(credentialsId: 'dockerhub', variable: 'DOCKERHUB_PASSWORD')]) {
                             sh "echo $DOCKERHUB_PASSWORD | docker login -u yourusername --password-stdin"
                             sh 'docker push yourusername/myapp:latest'
                         }
                     }
                 }
             }
             
             stage('Update Deployment YAML') {
                 steps {
                     script {
                         sh 'sed -i "s|image:.*|image: yourusername/myapp:latest|" k8s/deployment.yml'
                         git credentialsId: 'github-creds', url: 'https://github.com/yourusername/ci-cd-pipeline-demo.git'
                         sh 'git add k8s/deployment.yml'
                         sh 'git commit -m "Update deployment.yml with new Docker image"'
                         sh 'git push'
                     }
                 }
             }
         }
     }
     ```
   - Replace `yourusername` with your actual DockerHub and GitHub username.

4. **Configure SonarQube**:
   - Under "Manage Jenkins" > "Configure System", add your SonarQube server details.
   - Make sure you have a project on SonarQube corresponding to your repository.

5. **Run the Jenkins Pipeline**:
   - Once your pipeline is set up, trigger a build to ensure it successfully completes the Maven build, SonarQube analysis, Docker image creation, and push.

---

### **Step 3: Configure ArgoCD for Continuous Deployment**
1. **Install ArgoCD**:
   - Ensure ArgoCD is installed on your Kubernetes cluster and can sync with your GitHub repository.

2. **Create Kubernetes Deployment YAML**:
   - In your GitHub repo, create a folder `k8s/` and add a `deployment.yml` file for your application:
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
   - Commit and push the file.

3. **Create an ArgoCD Application**:
   - In ArgoCD UI, create a new application:
     - **Repository URL**: Your GitHub repository URL.
     - **Path**: `k8s/`.
     - **Destination**: Your Kubernetes cluster.

4. **Sync ArgoCD**:
   - ArgoCD will automatically sync the application by deploying the updated Docker image whenever the `deployment.yml` file is updated.

---

### **Step 4: Test Your CI/CD Flow**
1. **Make a Code Change**:
   - Modify your `App.java` or any part of the code and commit to GitHub.

2. **Observe the CI/CD Flow**:
   - Jenkins will automatically trigger the pipeline, run Maven build, SonarQube analysis, create a new Docker image, push it to DockerHub, update `deployment.yml`, and push the changes.
   - ArgoCD will detect the change in the `deployment.yml` file and deploy the new version to Kubernetes.

---

This end-to-end project sets up a fully automated CI/CD flow that includes code quality checks, containerization, deployment, and continuous delivery using ArgoCD.

-----

Here's the additional content for `service.yml`, `ingress.yml`, and a simple Java application (`App.java`).

---

### **Step 1: Create `service.yml`**
This file will expose your application as a service inside Kubernetes.

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

- **Explanation**:
  - This service will forward traffic from port 80 (external) to port 8080 (where the app runs inside the container).
  - `ClusterIP` is the internal service type, which means it will only be accessible from within the cluster.

---

### **Step 2: Create `ingress.yml`**
This file will configure the Ingress, which allows external traffic to reach your application.

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

- **Explanation**:
  - The `Ingress` directs traffic to the `myapp-service` using `myapp.local` as the host. 
  - This would require DNS settings or editing the local `/etc/hosts` file to map `myapp.local` to the IP of the Kubernetes cluster.

---

### **Step 3: Create a Simple Java Application (`App.java`)**
Here’s a basic Java application that can be compiled using Maven and packaged into a Docker image.

```java
public class App {
    public static void main(String[] args) {
        System.out.println("Hello, World! Simple Java Application for CI/CD Pipeline.");
    }
}
```

- **Explanation**:
  - This is a simple `main` method that prints a message when the application starts.

---

### **Step 4: Create a Basic Test Class (`AppTest.java`)**

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

- **Explanation**:
  - A very basic test to verify the setup. The test always passes by asserting `true`.

---

### **Step 5: Update `pom.xml`**
Make sure the `pom.xml` includes dependencies to compile and run the tests for your Maven project.

Here’s an example of what your `pom.xml` file could look like:

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

---

### **Final Folder Structure**

```bash
ci-cd-pipeline-demo/
├── src/
│   ├── main/
│   │   └── java/
│   │       └── App.java
│   └── test/
│       └── java/
│           └── AppTest.java
├── k8s/
│   ├── deployment.yml
│   ├── service.yml
│   └── ingress.yml
├── Dockerfile
├── Jenkinsfile
└── pom.xml
```

---

### **Step 6: Deploy the Application**
1. **Deploy on Kubernetes**:
   - Apply the YAML files for the deployment, service, and ingress:
     ```bash
     kubectl apply -f k8s/deployment.yml
     kubectl apply -f k8s/service.yml
     kubectl apply -f k8s/ingress.yml
     ```

2. **Access the Application**:
   - If running locally (like with Minikube), expose the Ingress IP or domain by editing `/etc/hosts` to map `myapp.local` to your Minikube or Kubernetes cluster IP.
     ```bash
     echo "192.168.99.100 myapp.local" | sudo tee -a /etc/hosts
     ```
   - Open `http://myapp.local` in your browser, and you should see the app response.

---

This project now includes everything you need for your CI/CD pipeline, containerized deployment, and exposure to external traffic through a Kubernetes service and ingress.

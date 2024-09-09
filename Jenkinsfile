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

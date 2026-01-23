pipeline {
    agent any

    environment {
        KUBECONFIG = credentials('kubeconfig')
        IMAGE_NAME = "opensecurity/mobile-security-framework-mobsf:latest"
    }

    stages {

        stage('Clone Code') {
            steps {
                git 'https://github.com/Srshinde5512/mobile-security.git'
            }
        }

        stage('Build App') {
            steps {
                sh 'mvn clean package'
            }
        }

        stage('Build Docker Image') {
            steps {
                sh 'docker build -t $IMAGE_NAME .'
            }
        }

        stage('Push Image') {
            steps {
                sh 'docker push $IMAGE_NAME'
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                sh 'kubectl apply -f deployment.yaml'
            }
        }
    }
}
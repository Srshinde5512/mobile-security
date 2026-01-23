pipeline {
    agent any

    environment {
        KUBECONFIG = credentials('kubeconfig')
        IMAGE_NAME = "opensecurity/mobile-security-framework-mobsf:latest"
    }

    stages {

        // stage('Clone Code') {
        //     steps {
        //         git branch: 'main', url: 'https://github.com/Srshinde5512/mobile-security.git'
        //     }
        // }

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
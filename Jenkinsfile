pipeline {
    agent any

    environment {
        KUBECONFIG_FILE = credentials('kubeconfig')
        // Using your actual Docker Hub username
        IMAGE_NAME = "batataman26/mobile-security-framework-mobsf:${env.BUILD_ID}"
        SCANNER_HOME = tool 'SonarScanner' 
    }

    stages {
        stage('Clone Code') {
            steps {
                git branch: 'main', url: 'https://github.com/Srshinde5512/mobile-security.git'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                script {
                    // Generate coverage for SonarQube
                    sh "/home/sunbeam/.local/bin/poetry run pytest --cov=. --cov-report=xml || true"
                    
                    withSonarQubeEnv('sonar-server') {
                        sh "${SCANNER_HOME}/bin/sonar-scanner \
                          -Dsonar.projectKey=mobsf-project \
                          -Dsonar.python.coverage.reportPaths=coverage.xml"
                    }
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 1, unit: 'HOURS') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                sh 'docker build -t $IMAGE_NAME .'
            }
        }

        stage('Trivy Security Scan') {
            steps {
                // Scans the image we just built
                sh "trivy image --exit-code 1 --severity CRITICAL $IMAGE_NAME"
            }
        }

        stage('Push Image') {
            steps {
                script {
                    // This uses the credentials ID 'dockerhub-creds' you created in Jenkins
                    docker.withRegistry('https://index.docker.io/v1/', 'dockerhub-creds') {
                        sh "docker push $IMAGE_NAME"
                    }
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                script {
                    // Dynamically update deployment.yaml to use the new build version
                    sh "sed -i 's|batataman26/mobile-security-framework-mobsf:latest|${IMAGE_NAME}|g' deployment.yaml"
                    
                    sh 'kubectl --kubeconfig=$KUBECONFIG_FILE apply -f deployment.yaml'
                }
            }
        }
    }

    post {
        always {
            // Clean up the local image to save space on the Master node
            sh "docker rmi $IMAGE_NAME || true"
        }
    }
}
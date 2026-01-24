pipeline {
    agent any

    environment {
        // Ensure you've added 'kubeconfig' as a 'Secret File' in Jenkins credentials
        KUBECONFIG_FILE = credentials('kubeconfig')
        IMAGE_NAME = "batataman26/mobile-security-framework-mobsf:${env.BUILD_ID}"
        SCANNER_HOME = tool 'SonarScanner' // Must match the name in Jenkins Global Tool Config
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
                        // 1. Run tests and generate the 'coverage.xml' report
                       sh "/home/sunbeam/.local/bin/poetry run pytest --cov=. --cov-report=xml"
            
                        // 2. Run the Sonar Scanner
                       withSonarQubeEnv('sonar-server') {
                        // Tell SonarQube exactly where to find the report
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
                    // This waits for SonarQube to send a webhook back to Jenkins
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
                // Fails the build if "CRITICAL" vulnerabilities are found
                sh "trivy image --exit-code 1 --severity CRITICAL $IMAGE_NAME"
            }
        }

        stage('Push Image') {
            steps {
                // Note: You'll need docker.withRegistry or 'docker login' here
                sh "docker push $IMAGE_NAME"
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                // Use the kubeconfig file provided by Jenkins
                sh 'kubectl --kubeconfig=$KUBECONFIG_FILE apply -f deployment.yaml'
            }
        }
    }
}
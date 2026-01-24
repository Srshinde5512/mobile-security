pipeline {
    agent {
        kubernetes {
            yaml '''
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: jnlp
    image: jenkins/inbound-agent:3355.v388858a_47b_33-7
    resources:
      limits:
        memory: "2Gi"
        cpu: "1000m"
      requests:
        memory: "1Gi"
        cpu: "500m"
'''
        }
    }

    environment {
        KUBECONFIG_FILE = credentials('kubeconfig')
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
                    // Running with increased RAM and excluding the stuck file
                    withSonarQubeEnv('sonar-server') {
                        sh "${SCANNER_HOME}/bin/sonar-scanner \
                          -Dsonar.projectKey=mobsf-project \
                          -Dsonar.exclusions=**/axml.py,**/StaticAnalyzer/tools/** \
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
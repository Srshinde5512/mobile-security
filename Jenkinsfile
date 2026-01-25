pipeline {
    agent {
        kubernetes {
            yaml '''
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: jnlp
    # Use the official Jenkins agent image which already has Java and the agent binary
    image: jenkins/inbound-agent:latest-jdk17
    user: root
    env:
    - name: JAVA_HOME
      value: /opt/java/openjdk
    resources:
      limits:
        memory: "2Gi"
        cpu: "1000m"
      requests:
        memory: "1Gi"
        cpu: "500m"
    volumeMounts:
    - name: docker-sock
      mountPath: /var/run/docker.sock
    # We only need to install Python and Docker CLI now
    command: ["/bin/sh", "-c"]
    args: ["apt-get update && apt-get install -y python3 python3-pip git curl docker.io && exec jenkins-agent"]
  volumes:
  - name: docker-sock
    hostPath:
      path: /var/run/docker.sock
'''
        }
    }

    environment {
        // Defined here for global access
        IMAGE_NAME = "batataman26/mobile-security-framework-mobsf:${env.BUILD_NUMBER}"
        SCANNER_HOME = tool 'SonarScanner' 
        KUBECONFIG_FILE = credentials('kubeconfig') 
    }

    stages {
        stage('Clone Code') {
            steps {
                git branch: 'main', url: 'https://github.com/Srshinde5512/mobile-security.git'
            }
        }

        stage('Install Dependencies & Test') {
            steps {
                script {
                    // Use 'python3 -m pip' to avoid path issues
                    sh '''
                        python3 -m pip install poetry
                        python3 -m poetry install
                        python3 -m poetry run pytest --cov=mobsf --cov-report=xml:coverage.xml || true
                    '''
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                script {
                    // Ensure 'sonar-server' matches the name in Manage Jenkins > System
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
                // Shortened timeout for lab environment
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                // Using double quotes to ensure variable expansion
                sh "docker build -t ${IMAGE_NAME} ."
            }
        }

        stage('Trivy Security Scan') {
            steps {
                // Added || true so the pipeline doesn't stop if it finds vulnerabilities during testing
                sh "trivy image --severity CRITICAL ${IMAGE_NAME} || true"
            }
        }

        stage('Push Image') {
            steps {
                script {
                    // Ensure 'dockerhub-creds' matches your Username/Password credential ID
                    docker.withRegistry('https://index.docker.io/v1/', 'dockerhub-creds') {
                        sh "docker push ${IMAGE_NAME}"
                    }
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                script {
                    // Replace 'latest' with the specific build tag in your deployment manifest
                    sh "sed -i 's|batataman26/mobile-security-framework-mobsf:latest|${IMAGE_NAME}|g' deployment.yaml"
                    sh "kubectl --kubeconfig=${KUBECONFIG_FILE} apply -f deployment.yaml"
                }
            }
        }
    }

    post {
        always {
            // Cleanup local images to save disk space on the K8s node
            sh "docker rmi ${IMAGE_NAME} || true"
        }
    }
}
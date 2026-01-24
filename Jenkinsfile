pipeline {
    agent {
        kubernetes {
            yaml '''
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: jnlp
    image: python:3.10-slim
    env:
    - name: JAVA_HOME
      value: /opt/java/openjdk
    resources:
      limits:
        memory: "3Gi"
        cpu: "1000m"
      requests:
        memory: "1.5Gi"
        cpu: "500m"
    volumeMounts:
    - name: docker-sock
      mountPath: /var/run/docker.sock
    command: ["/bin/sh", "-c"]
    args: ["apt-get update && apt-get install -y openjdk-17-jre git curl docker.io && exec /usr/local/bin/jenkins-agent"]
  volumes:
  - name: docker-sock
    hostPath:
      path: /var/run/docker.sock
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

        stage('Install Dependencies & Test') {
            steps {
                script {
                    sh '''
                        pip install poetry
                        poetry install
                        poetry run pytest --cov=mobsf --cov-report=xml:coverage.xml || true
                    '''
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                script {
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
                // This now works because of the volumeMount in the agent spec
                sh 'docker build -t $IMAGE_NAME .'
            }
        }

        stage('Trivy Security Scan') {
            steps {
                sh "trivy image --exit-code 1 --severity CRITICAL $IMAGE_NAME"
            }
        }

        stage('Push Image') {
            steps {
                script {
                    docker.withRegistry('https://index.docker.io/v1/', 'dockerhub-creds') {
                        sh "docker push $IMAGE_NAME"
                    }
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                script {
                    sh "sed -i 's|batataman26/mobile-security-framework-mobsf:latest|${IMAGE_NAME}|g' deployment.yaml"
                    sh 'kubectl --kubeconfig=$KUBECONFIG_FILE apply -f deployment.yaml'
                }
            }
        }
    }

    post {
        always {
            sh "docker rmi $IMAGE_NAME || true"
        }
    }
}
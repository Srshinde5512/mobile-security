pipeline {
    agent {
        kubernetes {
            yaml '''
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: jnlp
    image: jenkins/inbound-agent:latest-jdk17
    # This is the key fix: running the container as root
    securityContext:
      runAsUser: 0
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
    command: ["/bin/sh", "-c"]
    # Using 'which' to find the agent location automatically
    args: ["apt-get update && apt-get install -y python3 python3-pip git curl docker.io && exec $(which jenkins-agent)"]
  volumes:
  - name: docker-sock
    hostPath:
      path: /var/run/docker.sock
'''
        }
    }

    environment {
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
                    sh '''
                        python3 -m pip install poetry --break-system-packages || pip install poetry
                        python3 -m poetry install
                        python3 -m poetry run pytest --cov=mobsf --cov-report=xml:coverage.xml || true
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
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                sh "docker build -t ${IMAGE_NAME} ."
            }
        }

        stage('Push Image') {
            steps {
                script {
                    docker.withRegistry('https://index.docker.io/v1/', 'dockerhub-creds') {
                        sh "docker push ${IMAGE_NAME}"
                    }
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                script {
                    sh "sed -i 's|batataman26/mobile-security-framework-mobsf:latest|${IMAGE_NAME}|g' deployment.yaml"
                    sh "kubectl --kubeconfig=${KUBECONFIG_FILE} apply -f deployment.yaml"
                }
            }
        }
    }

    post {
        always {
            script {
                // Safeguard against the MissingPropertyException
                try {
                    sh "docker rmi ${IMAGE_NAME} || true"
                } catch (Exception e) {
                    echo "Could not remove image, likely because it was never defined: ${e.message}"
                }
            }
        }
    }
}
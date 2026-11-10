cd /root/devsecops-project/application

cat > Jenkinsfile <<'EOF'
pipeline {
    agent any

    environment {
        AWS_DEFAULT_REGION = 'us-east-1'
        AWS_ACCOUNT_ID = '699038657587'
        ECR_REPOSITORY = 'digital-marketing'
        EKS_CLUSTER = 'devsecops-eks'
        SONAR_PROJECT_KEY = 'swiggy-app'
        ECR_REGISTRY = '699038657587.dkr.ecr.us-east-1.amazonaws.com'
        ECR_URI = '699038657587.dkr.ecr.us-east-1.amazonaws.com/digital-marketing'
    }

    stages {
        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }

        stage('Checkout Source Code') {
            steps {
                checkout scm
                script {
                    env.GIT_SHORT = sh(
                        script: 'git rev-parse --short HEAD',
                        returnStdout: true
                    ).trim()

                    env.IMAGE_TAG = "${BUILD_NUMBER}-${env.GIT_SHORT}"
                    env.IMAGE_NAME = "${env.ECR_URI}:${env.IMAGE_TAG}"

                    echo "Building image: ${env.IMAGE_NAME}"
                }
            }
        }

        stage('Install Dependencies and Build') {
            steps {
                sh '''
                    set -e
                    node --version
                    npm --version
                    npm ci
                    npm run build
                    test -f dist/index.html
                '''
            }
        }

        stage('SonarQube Analysis') {
            steps {
                script {
                    def scannerHome = tool 'sonar-scanner'

                    withSonarQubeEnv('sonar-server') {
                        sh """
                            ${scannerHome}/bin/sonar-scanner \
                              -Dsonar.projectKey=${SONAR_PROJECT_KEY} \
                              -Dsonar.projectName=Swiggy-Application \
                              -Dsonar.sources=src \
                              -Dsonar.exclusions=**/node_modules/**,**/dist/**
                        """
                    }
                }
            }
        }

        stage('SonarQube Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('OWASP Dependency Check') {
            steps {
                withCredentials([
                    string(credentialsId: 'NVD-key', variable: 'NVD_API_KEY')
                ]) {
                    sh '''
                        set -e
                        mkdir -p reports/dependency-check

                        /opt/dependency-check/bin/dependency-check.sh \
                          --project "Swiggy Application" \
                          --scan package.json \
                          --scan package-lock.json \
                          --format XML \
                          --out reports/dependency-check \
                          --data /var/lib/jenkins/dependency-check-data \
                          --nvdApiKey "$NVD_API_KEY"
                    '''
                }
            }
            post {
                always {
                    dependencyCheckPublisher(
                        pattern: 'reports/dependency-check/dependency-check-report.xml'
                    )
                }
            }
        }

        stage('Trivy Filesystem Scan') {
            steps {
                sh '''
                    set -e
                    trivy fs \
                      --scanners vuln,secret \
                      --severity HIGH,CRITICAL \
                      --exit-code 1 \
                      --skip-dirs node_modules \
                      --skip-dirs dist \
                      .
                '''
            }
        }

        stage('Docker Build') {
            steps {
                sh '''
                    set -e
                    docker build -t "$IMAGE_NAME" .
                '''
            }
        }

        stage('Trivy Docker Image Scan') {
            steps {
                sh '''
                    set -e
                    trivy image \
                      --severity HIGH,CRITICAL \
                      --exit-code 1 \
                      "$IMAGE_NAME"
                '''
            }
        }

        stage('Push Image to Amazon ECR') {
            steps {
                sh '''
                    set -e

                    aws sts get-caller-identity

                    aws ecr get-login-password \
                      --region "$AWS_DEFAULT_REGION" \
                    | docker login \
                      --username AWS \
                      --password-stdin "$ECR_REGISTRY"

                    docker push "$IMAGE_NAME"
                '''
            }
        }

        stage('Deploy to Amazon EKS') {
            steps {
                sh '''
                    set -e

                    aws eks update-kubeconfig \
                      --region "$AWS_DEFAULT_REGION" \
                      --name "$EKS_CLUSTER"

                    kubectl get nodes

                    sed "s|IMAGE_PLACEHOLDER|$IMAGE_NAME|g" \
                      kubernetes/deployment.yml \
                      > deployment-rendered.yml

                    kubectl apply -f deployment-rendered.yml
                    kubectl apply -f kubernetes/service.yml

                    kubectl rollout status deployment/swiggy-app \
                      --timeout=300s

                    kubectl get deployments
                    kubectl get pods -o wide
                    kubectl get services
                '''
            }
        }
    }

    post {
        always {
            archiveArtifacts(
                artifacts: 'reports/**/*,deployment-rendered.yml',
                allowEmptyArchive: true
            )
        }

        success {
            echo 'Pipeline completed successfully. Application deployment was verified.'
        }

        failure {
            echo 'Pipeline failed. Check the failed stage and its console output.'
        }
    }
}
EOF

pipeline {
    agent any

    environment {
        GIT_CREDENTIALS_ID = 'Git'
        REPO_URL = 'https://github.com/mohithkumar96/devops-project.git'
        BRANCH = 'main'
        KUBECONFIG_CRED = 'KUBE_CONFIG' // Jenkins credential ID for kubeconfig.yaml

        // Docker variables
        DOCKER_IMAGE = "mohithkumar96/devops-app"
        DOCKER_TAG = "${env.BUILD_NUMBER}"
    }

    stages {
        stage('Checkout Code') {
            steps {
                git branch: 'main', url: 'https://github.com/mohithkumar96/devops-project.git'
            }
        }

        stage('Terraform Static Analysis') {
            steps {
                echo 'Running Terraform static code analysis...'
                sh '''
                    if ! command -v tfsec &> /dev/null; then
                        echo "tfsec not installed. Skipping static analysis."
                    else
                        tfsec ./terraform
                    fi
                '''
            }
        }

        stage('Docker Build & Push') {
            steps {
                echo 'Building Docker image...'
                sh '''
                    if ! command -v docker &> /dev/null; then
                        echo "Docker CLI not found. Skipping Docker build."
                    else
                        docker build -t ${DOCKER_IMAGE} .
                        docker push ${DOCKER_IMAGE}
                    fi
                '''
            }
        }

        stage('Image Security Scan') {
            steps {
                echo 'Running Trivy image scan...'
                sh '''
                    if ! command -v trivy &> /dev/null; then
                        echo "Trivy not installed. Skipping scan."
                    else
                        trivy image --exit-code 1 ${DOCKER_IMAGE} || true
                    fi
                '''
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                echo 'Deploying to Kubernetes...'
                withCredentials([file(credentialsId: "${KUBECONFIG_CRED}", variable: 'KUBECONFIG')]) {
                    sh '''
                        kubectl apply -f k8s/deployment.yaml
                        kubectl rollout status deployment/devops-app
                    '''
                }
            }
        }

        stage('Rollback (Optional)') {
            steps {
                echo 'Rollback stage (manual or scripted)'
                // Example: sh 'kubectl rollout undo deployment/devops-app'
            }
        }
    }

    post {
        always {
            echo 'Cleaning workspace...'
            cleanWs()
        }
        success {
            echo 'Pipeline completed successfully!'
        }
        failure {
            echo 'Pipeline failed. Check logs.'
        }
    }
}

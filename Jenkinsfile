pipeline {
    agent any

    environment {
        // GitHub credentials (Jenkins Credentials ID)
        GIT_CREDENTIALS_ID = 'Git'
        REPO_URL = 'https://github.com/mohithkumar96/devops-project.git'
        BRANCH = 'main'

        // Docker variables
        DOCKER_IMAGE = "mohithkumar96/devops-app"
        DOCKER_TAG = "${env.BUILD_NUMBER}"

        // Kubernetes context
        KUBE_CONTEXT = 'your-kube-context'  // Add your kubeconfig credentials in Jenkins

        // Optional: Add registry credentials if needed
        DOCKER_CREDENTIALS_ID = 'dockerhub-cred'
    }

    stages {
        stage('Code Checkout') {
            steps {
                echo "Checking out branch ${BRANCH}"
                git branch: "${BRANCH}",
                    url: "${REPO_URL}",
                    credentialsId: "${GIT_CREDENTIALS_ID}"
            }
        }

        stage('Static Code Analysis') {
            steps {
                echo "Running Terraform static code analysis..."
                sh '''
                # Example: using tfsec
                tfsec ./terraform || true
                '''
            }
        }

        stage('Docker Build & Push') {
            steps {
                script {
                    echo "Building Docker image..."
                    sh "docker build -t ${DOCKER_IMAGE}:${DOCKER_TAG} ."
                    echo "Pushing Docker image..."
                    docker.withRegistry('', "${DOCKER_CREDENTIALS_ID}") {
                        sh "docker push ${DOCKER_IMAGE}:${DOCKER_TAG}"
                    }
                }
            }
        }

        stage('Image Security Scan') {
            steps {
                echo "Scanning Docker image with Trivy..."
                sh "trivy image ${DOCKER_IMAGE}:${DOCKER_TAG} || true"
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                echo "Deploying application to Kubernetes..."
                sh '''
                kubectl --context=${KUBE_CONTEXT} apply -f k8s/deployment.yaml
                kubectl --context=${KUBE_CONTEXT} rollout status deployment/my-app
                '''
            }
        }

        stage('Rollback Capability') {
            steps {
                echo "Rollback stage placeholder - implement as needed"
                // Example: helm rollback or kubectl rollback commands
            }
        }
    }

    post {
        success {
            echo "Pipeline completed successfully!"
        }
        failure {
            echo "Pipeline failed. Please check logs."
        }
        always {
            echo "Cleaning workspace..."
            cleanWs()
        }
    }
}

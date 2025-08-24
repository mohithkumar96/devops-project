pipeline {
    agent any

    environment {
        REPO_URL = 'https://github.com/mohithkumar96/devops-project.git'
        BRANCH_NAME = 'main'
        DOCKER_IMAGE = 'mohithkumar96/devops-app'
        REGISTRY_CREDENTIALS = 'dockerhub-creds' // Jenkins credentials ID
        KUBE_CONTEXT = 'your-kube-context'       // Set in Jenkins credentials
    }

    stages {

        stage('Code Checkout') {
            steps {
                echo "Checking out branch ${BRANCH_NAME}"
                git branch: "${BRANCH_NAME}", url: "${REPO_URL}"
                credentialsId: 'Git'
            }
        }

        stage('Static Code Analysis') {
            steps {
                echo "Scanning Terraform code with tfsec"
                sh 'tfsec terraform/'  // Make sure tfsec is installed
                // Optional: scan app code with SonarQube or ESLint
            }
        }

        stage('Docker Build & Push') {
            steps {
                script {
                    def commitHash = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
                    def tag = "${DOCKER_IMAGE}:${commitHash}"
                    
                    echo "Building Docker image ${tag}"
                    sh "docker build -t ${tag} ."
                    
                    echo "Logging in to Docker Hub"
                    withCredentials([usernamePassword(credentialsId: "${REGISTRY_CREDENTIALS}", usernameVariable: 'USER', passwordVariable: 'PASS')]) {
                        sh "echo $PASS | docker login -u $USER --password-stdin"
                    }
                    
                    echo "Pushing Docker image to registry"
                    sh "docker push ${tag}"
                }
            }
        }

        stage('Image Security Scanning') {
            steps {
                echo "Scanning Docker image for vulnerabilities"
                sh "trivy image ${DOCKER_IMAGE}:latest" // Make sure Trivy is installed
            }
        }

        stage('Deployment to Kubernetes') {
            steps {
                echo "Deploying to Kubernetes cluster"
                sh "kubectl --context ${KUBE_CONTEXT} apply -f k8s/"
            }
        }

        stage('Rollback Capability') {
            steps {
                echo "Rollback options available using Helm or Git tags"
                // Example Helm rollback:
                // sh "helm rollback my-release 1 --kube-context ${KUBE_CONTEXT}"
            }
        }
    }

    post {
        always {
            echo "Cleaning workspace"
            cleanWs()
        }
        success {
            echo "Pipeline completed successfully"
        }
        failure {
            echo "Pipeline failed"
        }
    }
}


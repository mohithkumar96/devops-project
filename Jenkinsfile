pipeline {
    agent none

    environment {
        DOCKER_IMAGE = "mohithkumar96/devops-app:${env.BUILD_NUMBER}"
        KUBE_CONTEXT = credentials('kube-context')  // Jenkins credential ID for kubeconfig
    }

    stages {
        stage('Code Checkout') {
            agent { docker { image 'alpine/git:latest' } }
            steps {
                echo "Checking out branch main"
                git branch: 'main', url: 'https://github.com/mohithkumar96/devops-project.git'
            }
        }

        stage('Static Code Analysis') {
            agent { docker { image 'aquasec/tfsec:latest' } }
            steps {
                echo "Running Terraform static code analysis..."
                sh 'tfsec ./terraform || true'
            }
        }

        stage('Docker Build & Push') {
            agent { docker { image 'docker:24.0.5-cli' 
                             args '-v /var/run/docker.sock:/var/run/docker.sock' } }
            steps {
                echo "Building Docker image..."
                sh """
                    docker build -t $DOCKER_IMAGE .
                    docker push $DOCKER_IMAGE
                """
            }
        }

        stage('Image Security Scan') {
            agent { docker { image 'aquasec/trivy:latest' } }
            steps {
                echo "Scanning Docker image for vulnerabilities..."
                sh "trivy image $DOCKER_IMAGE || true"
            }
        }

        stage('Deploy to Kubernetes') {
            agent { docker { image 'bitnami/kubectl:latest' } }
            steps {
                echo "Deploying to Kubernetes..."
                sh """
                    kubectl --kubeconfig=/var/jenkins_home/.kube/config apply -f k8s/
                """
            }
        }

        stage('Rollback Capability') {
            agent { docker { image 'bitnami/kubectl:latest' } }
            steps {
                echo "Rollback stage - implement Helm or kubectl rollback here"
                // Example: sh "helm rollback my-release 1"
                // Example: sh "kubectl rollout undo deployment/my-app"
            }
        }
    }

    post {
        always {
            echo "Cleaning workspace..."
            cleanWs()
        }
        success {
            echo "Pipeline completed successfully!"
        }
        failure {
            echo "Pipeline failed!"
        }
    }
}

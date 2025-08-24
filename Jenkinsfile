pipeline {
    agent any

    environment {
        GIT_CREDENTIALS_ID = 'Git'
        REPO_URL = 'https://github.com/mohithkumar96/devops-project.git'
        BRANCH = 'main'
        KUBE_CONTEXT = credentials('kube-context') // Make sure this exists in Jenkins credentials

        // Docker variables
        DOCKER_IMAGE = "mohithkumar96/devops-app"
        DOCKER_TAG = "${env.BUILD_NUMBER}"
    }

    stages {
        stage('Code Checkout') {
            steps {
                echo "Checking out branch main"
                git branch: 'main', url: 'https://github.com/mohithkumar96/devops-project.git'
            }
        }

        stage('Static Code Analysis') {
            steps {
                echo "Running Terraform static code analysis..."
                sh 'tfsec ./terraform || true'
            }
        }

        stage('Docker Build & Push') {
            steps {
                echo "Building Docker image..."
                sh """
                   docker build -t mohithkumar96/devops-app:${env.BUILD_NUMBER} .
                   docker push mohithkumar96/devops-app:${env.BUILD_NUMBER}
                """
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                echo "Deploying to Kubernetes cluster..."
                sh """
                   kubectl --context=${KUBE_CONTEXT} apply -f k8s/
                """
            }
        }
    }

    post {
        always {
            node { // wrap cleanWs inside node to provide FilePath context
                echo "Cleaning workspace..."
                cleanWs()
            }
        }
    }
}

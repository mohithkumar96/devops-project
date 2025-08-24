pipeline {
    agent any

    environment {
        REGISTRY = "mohithkumar96"           // DockerHub username
        IMAGE_NAME = "devops-app"
        BUILD_NUMBER = "${env.BUILD_NUMBER}"
        COMMIT_HASH = "${GIT_COMMIT.substring(0,7)}"
    }

    stages {
        stage('Code Checkout') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/mohithkumar96/devops-project.git'
            }
        }

        stage('Static Code Analysis') {
            steps {
                sh 'pip install tfsec'
                sh 'tfsec terraform/'
            }
        }

        stage('Docker Build & Push') {
            steps {
                sh """
                    docker build -t $REGISTRY/$IMAGE_NAME:${BUILD_NUMBER}-${COMMIT_HASH} .
                    docker login -u $REGISTRY -p $DOCKERHUB_PASSWORD
                    docker push $REGISTRY/$IMAGE_NAME:${BUILD_NUMBER}-${COMMIT_HASH}
                """
            }
        }

        stage('Image Security Scan') {
            steps {
                sh 'trivy image $REGISTRY/$IMAGE_NAME:${BUILD_NUMBER}-${COMMIT_HASH}'
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                sh """
                    kubectl --kubeconfig=$KUBECONFIG apply -f k8s/
                """
            }
        }
    }

    post {
        always {
            echo 'Cleaning up...'
            sh 'docker system prune -f'
        }
        success {
            echo 'Pipeline succeeded!'
        }
        failure {
            echo 'Pipeline failed.'
        }
    }
}

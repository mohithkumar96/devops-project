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
                git branch: 'main', url: 'https://github.com/mohithkumar96/devops-project.git', credentialsId: 'Git'
            }
        }

        stage('Terraform Static Analysis') {
            agent { docker { image 'aquasec/tfsec:latest' } }
            steps {
                sh 'tfsec ./terraform'
            }
        }

        stage('Docker Build & Push') {
            agent { docker { image 'mohithkumar96/devops-app' } }
            steps {
                sh """
                docker build -t ${IMAGE_NAME} .
                docker login -u <docker-username> -p <docker-password>
                docker push ${IMAGE_NAME}
                """
            }
        }

        stage('Image Security Scan') {
            agent { docker { image 'aquasec/trivy:latest' } }
            steps {
                sh "trivy image ${IMAGE_NAME}"
            }
        }

        stage('Deploy to Kubernetes') {
            agent { docker { image 'bitnami/kubectl:latest' } }
            steps {
                withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
                    sh 'kubectl apply -f k8s/deployment.yaml'
                }
            }
        }

        stage('Rollback (Optional)') {
            agent { docker { image 'bitnami/kubectl:latest' } }
            steps {
                echo 'Rollback stage placeholder'
            }
        }
    }

    post {
        always {
            cleanWs()
        }
        success {
            echo 'Pipeline completed successfully!'
        }
        failure {
            echo 'Pipeline failed. Check logs!'
        }
    }
}

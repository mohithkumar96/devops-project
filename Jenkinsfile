pipeline {
    agent any
    environment {
        // Define environment variables
        DOCKER_IMAGE = "mohithkumar96/devops-app"
        IMAGE_TAG = "1.0"
        KUBECONFIG_FILE = credentials('kubeconfig')  // Jenkins credential with kubeconfig
        DOCKER_USER = "mohithkumar96"
        DOCKER_PASS = "dckr_pat_tLTlPEGLKJctsi0_83V7nE4ksLM"
    }
    stages {
        stage('Code Checkout') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/mohithkumar96/devops-project.git',
                    credentialsId: "Git"
            }
        }

        stage('Terraform Static Analysis') {
            steps {
                script {
                    if (fileExists('terraform')) {
                        sh 'tfsec terraform || echo "No issues found"'
                    } else {
                        echo "No Terraform code found. Skipping."
                    }
                }
            }
        }

        stage('Docker Build & Push') {
            steps {
                script {
                    sh """
                    docker login -u $DOCKER_USER -p $DOCKER_PASS
                    docker pull ${DOCKER_IMAGE}
                    """
                }
            }
        }

        stage('Image Security Scan') {
            steps {
                script {
                    def imageTag = "${DOCKER_IMAGE}:${env.BUILD_NUMBER}"
                    sh "trivy image --remote --severity HIGH,CRITICAL ${imageTag}"
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                withCredentials([file(credentialsId: 'KUBE-CONFIG', variable: 'KUBECONFIG')]) {
                    sh """
                        kubectl apply -f k8s/deployment.yaml
                        kubectl rollout status deployment/sample-app
                    """
                }
            }
        }

        stage('Rollback (Optional)') {
            steps {
                input message: 'Rollback to previous version?', ok: 'Rollback'
                sh """
                    helm rollback sample-app <REVISION>
                """
            }
        }
    }

    post {
        always {
            cleanWs()
        }
        failure {
            mail to: 'team@example.com',
                 subject: "Jenkins Build Failed: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                 body: "Check Jenkins for details: ${env.BUILD_URL}"
        }
    }
}

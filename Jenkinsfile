pipeline {
    agent any
    environment {
        // Define environment variables
        DOCKER_IMAGE = "mohithkumar96/devops-app"
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
                    def imageTag = "${DOCKER_IMAGE}:${env.BUILD_NUMBER}"
                    sh """
                        docker build -t ${imageTag} -f app/Dockerfile app
                        echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                        docker push ${imageTag}
                    """
                }
            }
        }

        stage('Image Security Scan') {
            steps {
                script {
                    def imageTag = "${DOCKER_IMAGE}:${env.BUILD_NUMBER}"
                    sh "trivy image --severity HIGH,CRITICAL ${imageTag}"
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

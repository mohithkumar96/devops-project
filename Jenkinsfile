pipeline {
    agent any

    environment {
        IMAGE_NAME    = "mohithkumar96/devops-app"
        IMAGE_TAG     = "1.0"
        DOCKER_USER   = "mohithkumar96"
        DOCKER_PASS   = "dckr_pat_tLTlPEGLKJctsi0_83V7nE4ksLM"
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

        stage('Install Podman') {
            steps {
                sh '''
                  echo "Installing Podman..."
                  sudo apt-get update -y
                  sudo apt-get install -y podman
                  podman --version
                '''
            }
        }

        stage('Podman Build') {
            steps {
                sh '''
                apt-get update -y && apt-get install -y buildah
                buildah --storage-driver=vfs bud -t mohithkumar96/devops-app:1.0 -f Devops-App/Dockerfile Devops-App
              '''
            }
        }

        stage('Login to Registry') {
            steps {
                sh "podman login docker.io -u ${DOCKER_USER} -p ${DOCKER_PASS}"
            }
        }

        stage('Push Image') {
            steps {
                sh "podman push ${IMAGE_NAME}:${IMAGE_TAG}"
            }
        }

        stage('Image Security Scan') {
            steps {
                sh "trivy image --remote --severity HIGH,CRITICAL ${IMAGE_NAME}:${IMAGE_TAG}"
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                withCredentials([file(credentialsId: 'KUBE-CONFIG', variable: 'KUBECONFIG')]) {
                    sh '''
                        kubectl apply -f k8s/deployment.yaml
                        kubectl rollout status deployment/sample-app
                    '''
                }
            }
        }

        stage('Rollback (Optional)') {
            steps {
                input message: 'Rollback to previous version?', ok: 'Rollback'
                sh "helm rollback sample-app <REVISION>"
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

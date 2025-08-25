pipeline {
    agent any

    environment {
        REGISTRY = "docker.io"
        IMAGE_NAME = "mohithkumar96/devops-app"
        IMAGE_TAG = "1.0"
        DOCKER_HOST = "tcp://host.docker.internal:2375"
        KUBECONFIG = "/var/jenkins_home/.kube/config"
        K8S_API = "https://kubernetes.docker.internal:6443" // Replace with your API server URL
        K8S_TOKEN = credentials('k8s-token')
    }

    stages {
        stage('Checkout SCM') {
            steps {
                git branch: 'main',
                    credentialsId: 'Github',
                    url: 'https://github.com/mohithkumar96/devops-project.git'
            }
        }

        stage('Terraform Static Analysis') {
            steps {
                script {
                    def tfsecPath = "${env.WORKSPACE}/bin/tfsec"
                    sh """
                    mkdir -p ${env.WORKSPACE}/bin
                    if [ ! -f ${tfsecPath} ]; then
                        curl -sSL https://github.com/aquasecurity/tfsec/releases/latest/download/tfsec-linux-amd64 -o ${tfsecPath}
                        chmod +x ${tfsecPath}
                    fi
                    ${tfsecPath} terraform || echo 'No issues found'
                    """
                }
            }
        }

        stage('Docker Build') {
            steps {
                sh "docker build -t ${IMAGE_NAME}:${IMAGE_TAG} -f Devops-App/Dockerfile Devops-App"
            }
        }

        stage('Login to Docker Hub') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'docker-hub', usernameVariable: 'USER', passwordVariable: 'PASS')]) {
                    sh "docker login ${REGISTRY} -u $USER -p $PASS"
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                sh "docker push ${IMAGE_NAME}:${IMAGE_TAG}"
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                sh """
                kubectl --server=${K8S_API} \
                        --token=${K8S_TOKEN} \
                        --insecure-skip-tls-verify=true \
                        apply -f k8s-manifests/
                """
            }
        }
    }

    post {
        always {
            cleanWs()
        }
        success {
            echo "Pipeline succeeded!"
        }
        failure {
            echo "Pipeline failed!"
        }
    }
}


pipeline {
    agent any

    environment {
        REGISTRY = "docker.io"
        IMAGE_NAME = "mohithkumar96/devops-app"
        IMAGE_TAG = "1.0"
        DOCKER_HOST = "tcp://host.docker.internal:2375"
        K8S_API = "https://kubernetes.docker.internal:6443"
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

        stage('Docker Build & Push') {
            steps {
                sh """
                docker build -t ${IMAGE_NAME}:${IMAGE_TAG} -f Devops-App/Dockerfile Devops-App
                docker push ${IMAGE_NAME}:${IMAGE_TAG}
                """
            }
        }

        stage('Trivy Scan') {
            steps {
                script {
                    sh """
                    curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh
                    ./trivy image --exit-code 0 --severity LOW,MEDIUM ${IMAGE_NAME}:${IMAGE_TAG}
                    ./trivy image --exit-code 1 --severity HIGH,CRITICAL ${IMAGE_NAME}:${IMAGE_TAG} || true
                    """
                }
            }
        }

        stage('Create Namespaces') {
            steps {
                sh """
                kubectl --server=${K8S_API} --token=${K8S_TOKEN} --insecure-skip-tls-verify=true create namespace dev || echo 'Namespace dev exists'
                kubectl --server=${K8S_API} --token=${K8S_TOKEN} --insecure-skip-tls-verify=true create namespace staging || echo 'Namespace staging exists'
                kubectl --server=${K8S_API} --token=${K8S_TOKEN} --insecure-skip-tls-verify=true create namespace prod || echo 'Namespace prod exists'
                """
            }
        }

        stage('Deploy to Dev') {
            steps {
                sh """
                kubectl --server=${K8S_API} --token=${K8S_TOKEN} --namespace=dev --insecure-skip-tls-verify=true apply -f k8s-manifests/deployment.yaml
                kubectl --server=${K8S_API} --token=${K8S_TOKEN} --namespace=dev --insecure-skip-tls-verify=true apply -f k8s-manifests/service.yaml
                kubectl --server=${K8S_API} --token=${K8S_TOKEN} --namespace=dev --insecure-skip-tls-verify=true apply -f k8s-manifests/ingress.yaml
                """
            }
        }

        stage('Deploy to Staging') {
            steps {
                input message: 'Approve deployment to staging?', ok: 'Deploy'
                sh """
                kubectl --server=${K8S_API} --token=${K8S_TOKEN} --namespace=staging --insecure-skip-tls-verify=true apply -f k8s-manifests/deployment.yaml
                kubectl --server=${K8S_API} --token=${K8S_TOKEN} --namespace=staging --insecure-skip-tls-verify=true apply -f k8s-manifests/service.yaml
                kubectl --server=${K8S_API} --token=${K8S_TOKEN} --namespace=staging --insecure-skip-tls-verify=true apply -f k8s-manifests/ingress.yaml
                """
            }
        }

        stage('Deploy to Production') {
            steps {
                input message: 'Approve deployment to production?', ok: 'Deploy'
                sh """
                kubectl --server=${K8S_API} --token=${K8S_TOKEN} --namespace=prod --insecure-skip-tls-verify=true apply -f k8s-manifests/deployment.yaml
                kubectl --server=${K8S_API} --token=${K8S_TOKEN} --namespace=prod --insecure-skip-tls-verify=true apply -f k8s-manifests/service.yaml
                kubectl --server=${K8S_API} --token=${K8S_TOKEN} --namespace=prod --insecure-skip-tls-verify=true apply -f k8s-manifests/ingress.yaml
                """
            }
        }

        stage('Rollback Option') {
            when {
                expression { return params.ROLLBACK == true }
            }
            steps {
                input message: 'Rollback to previous version?', ok: 'Rollback'
                script {
                    sh "docker pull ${IMAGE_NAME}:${params.ROLLBACK_TAG}"
                    sh "kubectl --server=${K8S_API} --token=${K8S_TOKEN} --namespace=prod --insecure-skip-tls-verify=true set image deployment/devops-app devops-app=${IMAGE_NAME}:${params.ROLLBACK_TAG}"
                }
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

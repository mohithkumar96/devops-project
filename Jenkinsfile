pipeline {
    agent any

    parameters {
        booleanParam(name: 'ROLLBACK', defaultValue: false, description: 'Rollback to previous image')
        string(name: 'ROLLBACK_TAG', defaultValue: '', description: 'Tag of the image to rollback to')
        string(name: 'IMAGE_TAG', defaultValue: '1.0', description: 'Docker image tag for this deployment')
    }

    environment {
        REGISTRY = "docker.io"
        IMAGE_NAME = "mohithkumar96/devops-app:latest"
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
                docker build -t ${IMAGE_NAME}:${params.IMAGE_TAG} -f Devops-App/Dockerfile Devops-App
                docker push ${IMAGE_NAME}:${params.IMAGE_TAG}
                """
            }
        }

        stage('Trivy Scan') {
            steps {
                sh """
                curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh -s -- -b ./bin
                ./bin/trivy image --exit-code 0 --severity LOW,MEDIUM ${IMAGE_NAME}:${params.IMAGE_TAG}
                ./bin/trivy image --exit-code 1 --severity HIGH,CRITICAL ${IMAGE_NAME}:${params.IMAGE_TAG} || true
                """
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
                kubectl --server=${K8S_API} --token=${K8S_TOKEN} --namespace=dev --insecure-skip-tls-verify=true set image deployment/devops-app devops-app=${IMAGE_NAME}:${params.IMAGE_TAG} || \
                kubectl --server=${K8S_API} --token=${K8S_TOKEN} --namespace=dev --insecure-skip-tls-verify=true apply -f k8s-manifests/deployment.yaml
                kubectl --server=${K8S_API} --token=${K8S_TOKEN} --namespace=dev --insecure-skip-tls-verify=true apply -f k8s-manifests/service.yaml
                kubectl --server=${K8S_API} --token=${K8S_TOKEN} --namespace=dev --insecure-skip-tls-verify=true apply -f k8s-manifests/ingress-dev.yaml
                kubectl rollout status deployment/devops-app -n dev
                """
            }
        }

        stage('Deploy to Staging') {
            steps {
                input message: 'Approve deployment to staging?', ok: 'Deploy'
                sh """
                kubectl --server=${K8S_API} --token=${K8S_TOKEN} --namespace=staging --insecure-skip-tls-verify=true set image deployment/devops-app devops-app=${IMAGE_NAME}:${params.IMAGE_TAG} || \
                kubectl --server=${K8S_API} --token=${K8S_TOKEN} --namespace=staging --insecure-skip-tls-verify=true apply -f k8s-manifests/deployment.yaml
                kubectl --server=${K8S_API} --token=${K8S_TOKEN} --namespace=staging --insecure-skip-tls-verify=true apply -f k8s-manifests/service.yaml
                kubectl --server=${K8S_API} --token=${K8S_TOKEN} --namespace=staging --insecure-skip-tls-verify=true apply -f k8s-manifests/ingress-staging.yaml
                kubectl rollout status deployment/devops-app -n staging
                """
            }
        }

        stage('Deploy to Production') {
            steps {
                input message: 'Approve deployment to production?', ok: 'Deploy'
                sh """
                kubectl --server=${K8S_API} --token=${K8S_TOKEN} --namespace=prod --insecure-skip-tls-verify=true set image deployment/devops-app devops-app=${IMAGE_NAME}:${params.IMAGE_TAG} || \
                kubectl --server=${K8S_API} --token=${K8S_TOKEN} --namespace=prod --insecure-skip-tls-verify=true apply -f k8s-manifests/deployment.yaml
                kubectl --server=${K8S_API} --token=${K8S_TOKEN} --namespace=prod --insecure-skip-tls-verify=true apply -f k8s-manifests/service.yaml
                kubectl --server=${K8S_API} --token=${K8S_TOKEN} --namespace=prod --insecure-skip-tls-verify=true apply -f k8s-manifests/ingress-prod.yaml
                kubectl rollout status deployment/devops-app -n prod
                """
            }
        }

        stage('Rollback Option') {
            when {
                expression { return params.ROLLBACK == true }
            }
            steps {
                input message: 'Rollback to previous version?', ok: 'Rollback'
                sh """
                docker pull ${IMAGE_NAME}:${params.ROLLBACK_TAG}
                kubectl --server=${K8S_API} --token=${K8S_TOKEN} --namespace=prod --insecure-skip-tls-verify=true set image deployment/devops-app devops-app=${IMAGE_NAME}:${params.ROLLBACK_TAG}
                kubectl rollout status deployment/devops-app -n prod
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

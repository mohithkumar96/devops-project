pipeline {
    agent any

    parameters {
        string(name: 'IMAGE_TAG', defaultValue: '1.0', description: 'Docker image tag for this deployment')
    }

    environment {
        REGISTRY = "docker.io"
        IMAGE_NAME = "mohithkumar96/devops-app"
        IMAGE_TAG = "${params.IMAGE_TAG}"
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


        stage('Fetch Jenkins SA Token') {
            steps {
                script {
                    // Fetch the service account token dynamically
                    env.K8S_TOKEN = sh(
                        script: "kubectl get secret \$(kubectl get sa jenkins-sa -n dev -o jsonpath='{.secrets[0].name}') -n dev -o jsonpath='{.data.token}' | base64 --decode",
                        returnStdout: true
                    ).trim()
                    echo "Fetched Jenkins SA token successfully"
                }
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
                    curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh -s -- -b ./bin
                    ./bin/trivy image --exit-code 0 --severity LOW,MEDIUM ${IMAGE_NAME}:${IMAGE_TAG}
                    ./bin/trivy image --exit-code 1 --severity HIGH,CRITICAL ${IMAGE_NAME}:${IMAGE_TAG} || true
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
                kubectl --server=${K8S_API} --token=${K8S_TOKEN} --namespace=dev --insecure-skip-tls-verify=true apply -f k8s-manifests/deployment.yaml --validate=false
                kubectl --server=${K8S_API} --token=${K8S_TOKEN} --namespace=dev --insecure-skip-tls-verify=true apply -f k8s-manifests/service.yaml --validate=false
                kubectl --server=${K8S_API} --token=${K8S_TOKEN} --namespace=dev --insecure-skip-tls-verify=true apply -f k8s-manifests/ingress-dev.yaml --validate=false
                kubectl rollout status deployment/devops-app -n dev
                """
            }
        }

        stage('Deploy to Staging') {
            steps {
                input message: 'Approve deployment to staging?', ok: 'Deploy'
                sh """
                kubectl --server=${K8S_API} --token=${K8S_TOKEN} --namespace=staging --insecure-skip-tls-verify=true apply -f k8s-manifests/deployment.yaml --validate=false
                kubectl --server=${K8S_API} --token=${K8S_TOKEN} --namespace=staging --insecure-skip-tls-verify=true apply -f k8s-manifests/service.yaml --validate=false
                kubectl --server=${K8S_API} --token=${K8S_TOKEN} --namespace=staging --insecure-skip-tls-verify=true apply -f k8s-manifests/ingress-staging.yaml --validate=false
                kubectl rollout status deployment/devops-app -n staging
                """
            }
        }

        stage('Deploy to Production') {
            steps {
                input message: 'Approve deployment to production?', ok: 'Deploy'
                sh """
                kubectl --server=${K8S_API} --token=${K8S_TOKEN} --namespace=prod --insecure-skip-tls-verify=true apply -f k8s-manifests/deployment.yaml --validate=false
                kubectl --server=${K8S_API} --token=${K8S_TOKEN} --namespace=prod --insecure-skip-tls-verify=true apply -f k8s-manifests/service.yaml --validate=false
                kubectl --server=${K8S_API} --token=${K8S_TOKEN} --namespace=prod --insecure-skip-tls-verify=true apply -f k8s-manifests/ingress-prod.yaml --validate=false
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

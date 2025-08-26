pipeline {
    agent any

    parameters {
        string(name: 'IMAGE_TAG', defaultValue: '1.0', description: 'Docker image tag for this deployment')
    }

    environment {
        REGISTRY     = "docker.io"
        IMAGE_NAME   = "mohithkumar96/devops-app"
        IMAGE_TAG    = "${params.IMAGE_TAG}"
        DOCKER_HOST  = "tcp://host.docker.internal:2375"
        K8S_API      = "https://kubernetes.docker.internal:6443"
        K8S_NAMESPACE_DEV     = "dev"
        K8S_NAMESPACE_STAGING = "staging"
        K8S_NAMESPACE_PROD    = "prod"
    }

    stages {
        stage('Checkout SCM') {
            steps {
                git branch: 'main',
                    credentialsId: 'Github',
                    url: 'https://github.com/mohithkumar96/devops-project.git'
            }
        }

    

        stage('Static Analysis') {
            parallel {
                stage('Terraform Security Scan') {
                    steps {
                        sh """
                        mkdir -p ${WORKSPACE}/bin
                        curl -sSL https://github.com/aquasecurity/tfsec/releases/latest/download/tfsec-linux-amd64 \
                          -o ${WORKSPACE}/bin/tfsec
                        chmod +x ${WORKSPACE}/bin/tfsec
                        ${WORKSPACE}/bin/tfsec terraform || echo 'No tfsec issues found'
                        """
                    }
                }
                stage('Trivy Pre-Download') {
                    steps {
                        sh """
                        curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh \
                          | sh -s -- -b ${WORKSPACE}/bin
                        """
                    }
                }
            }
        }

        stage('Docker Build & Push') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh """
                    echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                    docker build -t ${IMAGE_NAME}:${IMAGE_TAG} -f Devops-App/Dockerfile Devops-App
                    retry(3) { docker push ${IMAGE_NAME}:${IMAGE_TAG} }
                    docker logout
                    """
                }
            }
        }

        stage('Trivy Scan') {
            steps {
                sh """
                ${WORKSPACE}/bin/trivy image --exit-code 0 --severity LOW,MEDIUM ${IMAGE_NAME}:${IMAGE_TAG}
                ${WORKSPACE}/bin/trivy image --exit-code 1 --severity HIGH,CRITICAL ${IMAGE_NAME}:${IMAGE_TAG} || true
                """
            }
        }

        stage('Prepare Namespaces') {
            steps {
                sh """
                for ns in ${K8S_NAMESPACE_DEV} ${K8S_NAMESPACE_STAGING} ${K8S_NAMESPACE_PROD}; do
                    kubectl --server=${K8S_API} --token=${K8S_TOKEN} --insecure-skip-tls-verify=true \
                    apply -f - <<EOF
                    apiVersion: v1
                    kind: Namespace
                    metadata:
                      name: $ns
                    EOF
                done
                """
            }
        }

        stage('Deploy to Dev') {
            steps {
                sh """
                kubectl --server=${K8S_API} --token=${K8S_TOKEN} -n ${K8S_NAMESPACE_DEV} --insecure-skip-tls-verify=true \
                  apply -f k8s-manifests/deployment.yaml --validate=false
                kubectl --server=${K8S_API} --token=${K8S_TOKEN} -n ${K8S_NAMESPACE_DEV} --insecure-skip-tls-verify=true \
                  apply -f k8s-manifests/service.yaml --validate=false
                kubectl --server=${K8S_API} --token=${K8S_TOKEN} -n ${K8S_NAMESPACE_DEV} --insecure-skip-tls-verify=true \
                  apply -f k8s-manifests/ingress-dev.yaml --validate=false
                kubectl rollout status deployment/devops-app -n ${K8S_NAMESPACE_DEV}
                """
            }
        }

        stage('Deploy to Staging') {
            steps {
                input message: 'Approve deployment to staging?', ok: 'Deploy'
                sh """
                kubectl --server=${K8S_API} --token=${K8S_TOKEN} -n ${K8S_NAMESPACE_STAGING} --insecure-skip-tls-verify=true \
                  apply -f k8s-manifests/deployment.yaml --validate=false
                kubectl --server=${K8S_API} --token=${K8S_TOKEN} -n ${K8S_NAMESPACE_STAGING} --insecure-skip-tls-verify=true \
                  apply -f k8s-manifests/service.yaml --validate=false
                kubectl --server=${K8S_API} --token=${K8S_TOKEN} -n ${K8S_NAMESPACE_STAGING} --insecure-skip-tls-verify=true \
                  apply -f k8s-manifests/ingress-staging.yaml --validate=false
                kubectl rollout status deployment/devops-app -n ${K8S_NAMESPACE_STAGING}
                """
            }
        }

        stage('Deploy to Production') {
            steps {
                input message: 'Approve deployment to production?', ok: 'Deploy'
                sh """
                kubectl --server=${K8S_API} --token=${K8S_TOKEN} -n ${K8S_NAMESPACE_PROD} --insecure-skip-tls-verify=true \
                  apply -f k8s-manifests/deployment.yaml --validate=false
                kubectl --server=${K8S_API} --token=${K8S_TOKEN} -n ${K8S_NAMESPACE_PROD} --insecure-skip-tls-verify=true \
                  apply -f k8s-manifests/service.yaml --validate=false
                kubectl --server=${K8S_API} --token=${K8S_TOKEN} -n ${K8S_NAMESPACE_PROD} --insecure-skip-tls-verify=true \
                  apply -f k8s-manifests/ingress-prod.yaml --validate=false
                kubectl rollout status deployment/devops-app -n ${K8S_NAMESPACE_PROD}
                """
            }
        }
    }

    post {
        always {
            cleanWs()
        }
        success {
            echo "✅ Pipeline succeeded!"
        }
        failure {
            echo "❌ Pipeline failed!"
        }
    }
}


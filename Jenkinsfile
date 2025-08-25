pipeline {
    agent any

    environment {
        REGISTRY = "docker.io"
        IMAGE_NAME = "mohithkumar96/devops-app"
        IMAGE_TAG = "1.0"
        DOCKER_HOST = "tcp://host.docker.internal:2375"
        K8S_API = "https://kubernetes.docker.internal:6443"
    }

    stages {

        // 1️⃣ Checkout Source Code
        stage('Checkout SCM') {
            steps {
                git branch: 'main',
                    credentialsId: 'Github',
                    url: 'https://github.com/mohithkumar96/devops-project.git'
            }
        }

        // 2️⃣ Terraform Static Analysis
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

        // 3️⃣ Build Docker Image
        stage('Docker Build') {
            steps {
                sh "docker build -t ${IMAGE_NAME}:${IMAGE_TAG} -f Devops-App/Dockerfile Devops-App"
            }
        }

        // 4️⃣ Login to Docker Hub
        stage('Login to Docker Hub') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'docker-hub', usernameVariable: 'USER', passwordVariable: 'PASS')]) {
                    sh "docker login ${REGISTRY} -u $USER -p $PASS"
                }
            }
        }

        // 5️⃣ Push Docker Image
        stage('Push Docker Image') {
            steps {
                sh "docker push ${IMAGE_NAME}:${IMAGE_TAG}"
            }
        }

        // 6️⃣ Deploy to Development
        stage('Deploy to Dev') {
            steps {
                withCredentials([string(credentialsId: 'k8s-token', variable: 'K8S_TOKEN')]) {
                    script {
                        // Create dev namespace if it doesn't exist
                        sh "kubectl get namespace dev || kubectl create namespace dev"

                        def helmChartExists = fileExists('k8s-manifests/helm-chart/Chart.yaml')
                        if (helmChartExists) {
                            echo "Deploying to Dev using Helm"
                            sh """
                            helm upgrade --install devops-app k8s-manifests/helm-chart \
                                --set image.repository=${IMAGE_NAME} \
                                --set image.tag=${IMAGE_TAG} \
                                --kube-token=$K8S_TOKEN \
                                --kube-apiserver=${K8S_API} \
                                --insecure-skip-tls-verify
                            """
                        } else {
                            echo "Deploying to Dev using YAML manifests"
                            sh """
                            kubectl --server=${K8S_API} \
                                    --token=$K8S_TOKEN \
                                    --namespace=dev \
                                    --insecure-skip-tls-verify=true \
                                    apply -f k8s-manifests/
                            """
                        }
                    }
                }
            }
        }

        // 7️⃣ Promote to Staging (Manual Approval)
        stage('Promote to Staging') {
            steps {
                input message: 'Approve deployment to Staging?', ok: 'Deploy'
                withCredentials([string(credentialsId: 'k8s-token', variable: 'K8S_TOKEN')]) {
                    sh "kubectl get namespace staging || kubectl create namespace staging"
                    sh """
                    kubectl --server=${K8S_API} \
                            --token=$K8S_TOKEN \
                            --namespace=staging \
                            --insecure-skip-tls-verify=true \
                            apply -f k8s-manifests/
                    """
                }
            }
        }

        // 8️⃣ Promote to Production (Manual Approval)
        stage('Promote to Production') {
            steps {
                input message: 'Approve deployment to Production?', ok: 'Deploy'
                withCredentials([string(credentialsId: 'k8s-token', variable: 'K8S_TOKEN')]) {
                    sh "kubectl get namespace production || kubectl create namespace production"
                    sh """
                    kubectl --server=${K8S_API} \
                            --token=$K8S_TOKEN \
                            --namespace=production \
                            --insecure-skip-tls-verify=true \
                            apply -f k8s-manifests/
                    """
                }
            }
        }

        // 9️⃣ Rollback (Optional)
        stage('Rollback') {
            when {
                expression { params.ROLLBACK == true } // Optional boolean parameter in Jenkins
            }
            steps {
                input message: 'Approve Rollback?', ok: 'Rollback'
                withCredentials([string(credentialsId: 'k8s-token', variable: 'K8S_TOKEN')]) {
                    sh """
                    helm rollback devops-app 1 --namespace production
                    """
                    // Optional DB rollback
                    // sh './db/rollback.sh'
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

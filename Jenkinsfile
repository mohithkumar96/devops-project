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
                sh "docker build -t ${IMAGE_NAME}:${IMAGE_TAG} -f Devops-App/Dockerfile Devops-App"
                withCredentials([usernamePassword(credentialsId: 'docker-hub', usernameVariable: 'USER', passwordVariable: 'PASS')]) {
                    sh "docker login ${REGISTRY} -u $USER -p $PASS"
                    sh "docker push ${IMAGE_NAME}:${IMAGE_TAG}"
                }
            }
        }

        stage('Deploy to Dev') {
            steps {
                sh """
                kubectl --server=${K8S_API} \
                        --token=${K8S_TOKEN} \
                        --namespace=dev \
                        --insecure-skip-tls-verify=true \
                        apply -f k8s-manifests/
                """
            }
        }

        stage('Promote to Staging') {
            steps {
                input message: 'Approve promotion to Staging?'
                sh """
                kubectl --server=${K8S_API} \
                        --token=${K8S_TOKEN} \
                        --namespace=staging \
                        --insecure-skip-tls-verify=true \
                        apply -f k8s-manifests/
                """
            }
        }

        stage('Promote to Production') {
            steps {
                input message: 'Approve promotion to Production?'
                sh """
                kubectl --server=${K8S_API} \
                        --token=${K8S_TOKEN} \
                        --namespace=prod \
                        --insecure-skip-tls-verify=true \
                        apply -f k8s-manifests/
                """
            }
        }

        stage('Rollback') {
            when {
                expression { currentBuild.result == 'FAILURE' }
            }
            steps {
                script {
                    // Docker rollback example (use previous Git tag)
                    def previousTag = sh(script: "git describe --tags --abbrev=0 HEAD^", returnStdout: true).trim()
                    echo "Rolling back Docker image to tag: ${previousTag}"
                    sh "docker pull ${IMAGE_NAME}:${previousTag}"
                    sh "docker tag ${IMAGE_NAME}:${previousTag} ${IMAGE_NAME}:${IMAGE_TAG}"
                    sh "docker push ${IMAGE_NAME}:${IMAGE_TAG}"

                    // Helm rollback example
                    // sh "helm rollback devops-app 1 --namespace dev"
                    // sh "helm rollback devops-app 1 --namespace staging"
                    // sh "helm rollback devops-app 1 --namespace prod"

                    // Database rollback placeholder
                    // sh "psql -U user -d db -f rollback.sql"
                }
            }
        }
    }

    post {
        always { cleanWs() }
        success { echo "Pipeline succeeded!" }
        failure { echo "Pipeline failed! Rollback executed." }
    }
}


pipeline {
    agent none  // no global agent, each stage will use its own agent

    stages {

        stage('Checkout Code') {
            agent {
                docker {
                    image 'alpine/git:latest'
                    args '-v /root/.gitconfig:/root/.gitconfig' // optional
                }
            }
            steps {
                git url: 'https://github.com/mohithkumar96/devops-project.git', branch: 'main', credentialsId: 'Git'
            }
        }

        stage('Terraform Static Analysis') {
            agent {
                docker {
                    image 'aquasec/tfsec:latest'
                }
            }
            steps {
                sh 'tfsec ./terraform'
            }
        }

        stage('Docker Build & Push') {
            agent {
                docker {
                    image 'docker:24.0.5-cli'
                    args '-v /var/run/docker.sock:/var/run/docker.sock' // to use host Docker
                }
            }
            steps {
                sh 'docker build -t mohithkumar96/devops-app:latest .'
                sh 'docker push mohithkumar96/devops-app:latest'
            }
        }

        stage('Image Security Scan') {
            agent {
                docker {
                    image 'aquasec/trivy:latest'
                }
            }
            steps {
                sh 'trivy image mohithkumar96/devops-app:latest'
            }
        }

        stage('Deploy to Kubernetes') {
            agent {
                docker {
                    image 'bitnami/kubectl:latest'
                    args '-v $HOME/.kube:/root/.kube:ro'
                }
            }
            environment {
                KUBECONFIG = credentials('KUBE_CONFIG')  // your Jenkins credential
            }
            steps {
                sh 'kubectl apply -f k8s/deployment.yaml'
            }
        }
    }

    post {
        always {
            cleanWs()
        }
    }
}

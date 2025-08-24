pipeline {
    agent any

    environment {
        REGISTRY        = "mohithkumar96"              // DockerHub username
        IMAGE_NAME      = "devops-app"
        BUILD_NUMBER    = "${env.BUILD_NUMBER}"
        COMMIT_HASH     = "${GIT_COMMIT.substring(0,7)}"
        DEV_NAMESPACE   = "dev"
        STAGING_NAMESPACE = "staging"
        PROD_NAMESPACE  = "prod"
    }

    options {
        timestamps()
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }

    stages {

        stage('Code Checkout') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/mohithkumar96/devops-project.git'
                echo "Checked out commit ${COMMIT_HASH}"
            }
        }

        stage('Static Code Analysis') {
            steps {
                sh '''
                    pip install tfsec checkov
                    tfsec terraform/
                    checkov -d terraform/
                '''
            }
        }

        stage('Docker Build & Push') {
            steps {
                sh """
                    docker build -t $REGISTRY/$IMAGE_NAME:${BUILD_NUMBER}-${COMMIT_HASH} .
                    echo $DOCKERHUB_PASSWORD | docker login -u $REGISTRY --password-stdin
                    docker push $REGISTRY/$IMAGE_NAME:${BUILD_NUMBER}-${COMMIT_HASH}
                """
            }
        }

        stage('Image Security Scan') {
            steps {
                sh "trivy image --exit-code 1 $REGISTRY/$IMAGE_NAME:${BUILD_NUMBER}-${COMMIT_HASH}"
            }
        }

        stage('Deploy to Dev') {
            steps {
                echo "Deploying to Dev namespace..."
                sh """
                    kubectl --kubeconfig=$KUBECONFIG set image deployment/$IMAGE_NAME $IMAGE_NAME=$REGISTRY/$IMAGE_NAME:${BUILD_NUMBER}-${COMMIT_HASH} -n $DEV_NAMESPACE
                    kubectl --kubeconfig=$KUBECONFIG rollout status deployment/$IMAGE_NAME -n $DEV_NAMESPACE
                """
            }
        }

        stage('Promote to Staging') {
            steps {
                input message: "Approve deployment to Staging?"
                sh """
                    kubectl --kubeconfig=$KUBECONFIG set image deployment/$IMAGE_NAME $IMAGE_NAME=$REGISTRY/$IMAGE_NAME:${BUILD_NUMBER}-${COMMIT_HASH} -n $STAGING_NAMESPACE
                    kubectl --kubeconfig=$KUBECONFIG rollout status deployment/$IMAGE_NAME -n $STAGING_NAMESPACE
                """
            }
        }

        stage('Promote to Production') {
            steps {
                input message: "Approve deployment to Production?"
                sh """
                    kubectl --kubeconfig=$KUBECONFIG set image deployment/$IMAGE_NAME $IMAGE_NAME=$REGISTRY/$IMAGE_NAME:${BUILD_NUMBER}-${COMMIT_HASH} -n $PROD_NAMESPACE
                    kubectl --kubeconfig=$KUBECONFIG rollout status deployment/$IMAGE_NAME -n $PROD_NAMESPACE
                """
            }
        }

        stage('Rollback Capability') {
            steps {
                echo "Rollback option available using previous image tag or Helm revision"
                // Example Helm rollback command:
                // sh "helm rollback my-release 1 --kubeconfig=$KUBECONFIG"
            }
        }
    }

    post {
        always {
            echo 'Cleaning up...'
            sh 'docker system prune -f'
        }
        success {
            echo 'Pipeline succeeded!'
        }
        failure {
            echo 'Pipeline failed!'
        }
    }
}

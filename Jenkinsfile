pipeline {
    agent any

    environment {
        DOCKERHUB_USER = credentials('mohithkumar96')
        DOCKERHUB_PASS = credentials('Emohithkumar@1996')
    }

    stages {
        stage('Checkout') {
            steps { git 'https://github.com/mohithkumar96/devops-project.git' }
        }

        stage('Build Docker Image') {
            steps {
                sh 'docker build -t $DOCKERHUB_USER/devops-app:latest ./app'
            }
        }

        stage('Push to DockerHub') {
            steps {
                sh 'echo $DOCKERHUB_PASS | docker login -u $DOCKERHUB_USER --password-stdin'
                sh 'docker push $DOCKERHUB_USER/devops-app:latest'
            }
        }

        stage('Deploy to EKS') {
            steps {
                sh 'aws eks --region us-east-1 update-kubeconfig --name devops-eks'
                sh 'kubectl apply -f k8s/deployment.yaml'
                sh 'kubectl apply -f k8s/service.yaml'
            }
        }
    }
}

pipeline {
    agent none

    environment {
        AWS_REGION = "ap-south-1" ## replace with your aws-region
        ECR_REPO = "123456789012.dkr.ecr.ap-south-1.amazonaws.com/my-app" ## replace with demo-app-ecr-repo
        IMAGE_TAG = "${BUILD_NUMBER}"
        GIT_REPO = "https://github.com/username/repository.git" ## replace with demo-app

    stages {

        stage('Code Checkout') {
            agent any
            steps {
                echo "Cloning public repository..."
                git url: "${GIT_REPO}", branch: "devlop"
            }
        }

        stage('Build Application') {
            agent { label 'maven-agent' }

            steps {
                echo "Building application using Maven"
                sh 'mvn clean package -DskipTests'
            }
        }

        stage('Run Tests') {
            agent { label 'maven-agent' }

            steps {
                echo "Running tests"
                sh 'mvn test'
            }
        }

        stage('Build & Push Docker Image') {
            agent { label 'docker-node' }
 
            steps {
                script {

                    sh """
                    aws ecr get-login-password --region ${AWS_REGION} \
                    | docker login --username AWS --password-stdin ${ECR_REPO}
                    """

                    sh """
                    docker build -t my-app:${IMAGE_TAG} .
                    docker tag my-app:${IMAGE_TAG} ${ECR_REPO}:${IMAGE_TAG}
                    docker push ${ECR_REPO}:${IMAGE_TAG}
                    """
                }
            }
        }

        stage('Approval for Production') {
            steps {
                input message: "Deploy to Production EKS?", ok: "Deploy"
            }
        }

        stage('Deploy to EKS') {
            agent { label 'ansible-agent' }

            steps {
                echo "Deploying application to EKS using Ansible"

                sh """
                ansible-playbook deploy.yml \
                --extra-vars "image=${ECR_REPO}:${IMAGE_TAG}"
                """
            }
        }
    }

    post {
        success {
            echo "Pipeline completed successfully!"
        }

        failure {
            echo "Pipeline failed!"
        }
    }
}
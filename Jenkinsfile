pipeline {
    agent any
    tools {
        nodejs 'NodeJS' // Ensure NodeJS is configured in Jenkins
    }
    environment {
        SONAR_PROJECT_KEY = 'CICD-docker'
        SONAR_SCANNER_HOME = tool 'SonarQubeScanner'
        DOCKER_HUB_REPO = 'sampatel8543/cicd-project'
        JOB_NAME_NOW = 'cicd01'
        ECR_REPO = 'dev-repo'
        IMAGE_TAG = 'latest'
        ECR_REGISTRY = '905417999377.dkr.ecr.us-east-1.amazonaws.com'
        AWS_REGION = 'us-east-1'
    }
    stages {
        stage('Create ECR Repository') {
            steps {
                script {
                    echo "Checking if ECR repository exists or creating it..."
                    sh """
                    aws ecr describe-repositories --repository-names ${ECR_REPO} --region ${AWS_REGION} || \
                    aws ecr create-repository --repository-name ${ECR_REPO} --region ${AWS_REGION}
                    """
                }
            }
        }
        stage('GitHub Checkout') {
            steps {
                git branch: 'main', credentialsId: 'Alokm77', url: 'https://github.com/Alokm77/CICD-Docker.git'
            }
        }
        stage('Unit Test') {
            steps {
                sh 'npm install' // Install dependencies
                sh 'npm test'    // Run unit tests
            }
        }
        stage('SonarQube Analysis') {
            steps {
                withCredentials([string(credentialsId: 'CICD-Docker', variable: 'SONAR_TOKEN')]) {
                    withSonarQubeEnv('SonarQube') {
                        sh """
                        ${SONAR_SCANNER_HOME}/bin/sonar-scanner \
                        -Dsonar.projectKey=${SONAR_PROJECT_KEY} \
                        -Dsonar.sources=. \
                        -Dsonar.host.url=http://sonarqube-dev:9000 \
                        -Dsonar.login=${SONAR_TOKEN}
                        """
                    }
                }
            }
        }
        stage('Build Docker Image') {
            steps {
                script {
                    docker.build("${ECR_REGISTRY}/${ECR_REPO}:${IMAGE_TAG}")
                }
            }
        }
        stage('Trivy Scan') {
            steps {
                sh """
                trivy --severity HIGH,CRITICAL --no-progress --format table -o trivy-report.html image ${ECR_REGISTRY}/${ECR_REPO}:${IMAGE_TAG}
                """
            }
        }
        stage('Login to ECR') {
            steps {
                sh """
                aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_REGISTRY}
                """
            }
        }
        stage('Push Image to ECR') {
            steps {
                script {
                    docker.image("${ECR_REGISTRY}/${ECR_REPO}:${IMAGE_TAG}").push()
                }
            }
        }
    }
}

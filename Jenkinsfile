pipeline {
	agent any
	tools{
		nodejs 'NodeJS'
	}
	environment{
		SONAR_PROJECT_KEY = 'CICD-docker'
		SONAR_SCANNER_HOME = tool 'SonarQubeScanner'
		DOCKER_HUB_REPO = 'sampatel8543/cicd-project'
		JOB_NAME_NOW = 'cicd01'
		ECR_REPO = 'dev-repo'
		IMAGE_TAG = 'latest'
		ECR_REGISTRY = '905417999377.dkr.ecr.us-east-1.amazonaws.com'	
	}
	stages {
		stage('GitHub'){
			steps {
				git branch: 'main', credentialsId: 'Alokm77', url: 'https://github.com/Alokm77/CICD-Docker.git'
			}
		}
		stage('Unit Test'){
			steps {
				sh 'npm install' // Install dependencies
                		sh 'npm test'    // Run unit tests
			}
		}
		stage('SonarQube Analysis'){
			steps {
				withCredentials([string(credentialsId: 'CICD-Docker', variable: 'SONAR_TOKEN')]) {
    // some block

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
		stage('Docker Image'){
			steps {
				script {
					docker.build("${ECR_REGISTRY}/${ECR_REPO}:${IMAGE_TAG}")
				}
			}
		}
				stage('Trivy Scan'){
			steps {
				sh 'trivy --severity HIGH,CRITICAL --no-progress --format table -o trivy-report.html image ${JOB_NAME_NOW}:latest'
			
			}
		}
		stage('Login to ECR'){
			steps {
				sh """
				aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin 905417999377.dkr.ecr.us-east-1.amazonaws.com
				"""
			}
		}
		stage('Push Image to ECR'){
			steps {
				script {
				docker.image("${ECR_REGISTRY}/${ECR_REPO}:${IMAGE_TAG}").push()
				}
			}
		}
	}
}

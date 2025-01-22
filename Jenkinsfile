pipeline {
	agent any
	tools{
		nodejs 'NodeJS'
	}
	environment{
		SONAR_PROJECT_KEY = 'CICD-docker'
		SONAR_SCANNER_HOME = tool 'SonarQubeScanner'
		DOCKER_HUB_REPO = 'sampatel8543/cicd-project'
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
					docker.build("${DOCKER_HUB_REPO}:latest")
				}
			}
		}
	}
}

pipeline {
	agent any
	tools{
		nodejs 'NodeJS
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
	}
}

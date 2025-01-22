pipeline {
	agent any
	
	stages {
		stage('GitHub'){
			steps {
				git branch: 'main', credentialsId: 'Alokm77', url: 'https://github.com/Alokm77/CICD-Docker.git'
			}
		}
		stage('Unit Test'){
			step {
			}
		}
	}
}

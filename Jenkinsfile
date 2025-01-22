pipeline {
	agent ant
	
	stages {
		stage('GitHub'){
			steps {
				git branch: 'main', credentialsId: 'Alokm77', url: 'https://github.com/Alokm77/CICD-Docker.git'
			}
		}
	}
}

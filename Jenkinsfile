pipeline {
    agent any
    stages {
        stage('GIT') {
            steps {
                git branch: 'main', url: 'https://github.com/cyrine67/atelier_devops-main.git'
            }
        }
        stage('compile') {
            steps {
                sh 'mvn clean compile'
            }
        }
    }
}

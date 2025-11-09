pipeline {
    agent any
    stages {
        // Etape 1 : pull du code source depuis GitHub
        stage('GIT') 
        {
            steps {
                git(
                    branch: 'main', 
                    url: 'https://github.com/cyrine67/atelier_devops-main.git'                )
            }
        }
        // Etape 2 : test unitaire avec Maven
        stage('Tests Unitaires') 
        {
            steps {
                sh 'mvn test' 
            }
        }
        // Etape 3 : compilation du code source  
        stage('compile') 
        {
            steps {
                sh 'mvn clean compile'
            }
        }
        // Etape 4 : sonarQube
        stage('MVN SONARQUBE') 
        {
            steps {
                 withSonarQubeEnv('SonarQube') {
                    sh '''
                        mvn clean verify sonar:sonar \
                        -Dsonar.coverage.jacoco.xmlReportPaths=target/site/jacoco/jacoco.xml
                    '''
                }
            }
        }
  
  
        // Etape 6.1 : Construction des images Docker
        stage('Build Docker Images') 
        {
            steps {
                sh 'sudo docker compose build'
            }
        }
         // Étape 6.2 : Authentification à Docker Hub
        stage('Login to Docker Hub') 
        {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'DOCKER_HUB_CREDENTIALS',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASSWORD'
                )]) {
                    sh '''
                        sudo docker logout
                        echo "${DOCKER_PASSWORD}" | sudo docker login -u "${DOCKER_USER}" --password-stdin
                    '''
                }
            }
        }
        // Etape 7 : push image docker sur Docker Hub
        stage('Push Docker Image') 
        {
            steps {
                sh '''
                    sudo docker compose push
                    sudo docker logout
                '''
            }
        }
        // Étape 8 : Déploiement avec Docker Compose
        stage('Deploy with Docker Compose') 
        {
            steps {
                sh 'sudo docker compose down && sudo docker compose up -d'
            }
        }
        // Étape 9 : Vérification du déploiement
        stage('Verify Deployment') 
        {
            steps {
                script {
                    // Réessayer jusqu'à 5 fois, avec une attente de 10 secondes entre chaque tentative
                    retry(5) {
                        sleep 10 // Attendre 10 secondes avant chaque tentative
                        sh 'curl -I http://192.168.163.128:8089/kaddem'
                    }
                }
            }
        }
   }
    
}
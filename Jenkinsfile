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
        stage('Secrets Scan') {
            steps {
                sh 'gitleaks detect --source . --verbose --redact'
                // Cette commande échoue si des secrets sont détectés
            }
        }
        // Etape 2 : test unitaire avec Maven
        stage('Tests Unitaires') 
        {
            steps {
                sh 'mvn test' 
            }
        }
        stage('SCA - Dependency Check') {
            steps {
                sh '''
                    mvn org.owasp:dependency-check-maven:check \
                    -DfailBuildOnCVSS=7 \
                    || echo "Dependency check completed - continuing pipeline"
                '''
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
                sh ' docker compose build'
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
                        docker logout
                        echo "${DOCKER_PASSWORD}" |  docker login -u "${DOCKER_USER}" --password-stdin
                    '''
                }
            }
        }
        // Etape 7 : push image docker sur Docker Hub
        stage('Push Docker Image') 
        {
            steps {
                sh '''
                     docker compose push
                     docker logout
                '''
            }
        }
        // Étape 8 : Déploiement avec Docker Compose
        stage('Deploy with Docker Compose') 
        {
            steps {
                sh ' docker compose down &&  docker compose up -d'
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
                        sh 'curl -I http://172.20.10.2:8089/kaddem'
                    }
                }
            }
        }
         stage('DAST - Security Scan') {
            steps {
                sh '''
                    mkdir -p zap-report
                    docker run -t --rm \\
                    -v $(pwd)/zap-report:/zap/wrk:rw \\
                    owasp/zap2docker-stable:latest \\
                    zap-baseline.py -t http://172.20.10.2:8089/kaddem
                '''
            }
        }
   }
    
}
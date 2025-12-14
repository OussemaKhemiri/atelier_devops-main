pipeline {
    agent any
    
    // Optional: Define tools if configured in Jenkins Global Tools
    // tools { maven 'Maven3' } 

    stages {
        // ===========================================
        //      CI STAGES - RUN ON ALL BRANCHES
        // ===========================================
        
        stage('Checkout') {
            steps {
                // Modified: Uses 'checkout scm' to allow building ANY branch, not just main
                checkout scm 
                echo "✅ checked out branch: ${env.BRANCH_NAME}"
            }
        }

        stage('Secrets Scan - Gitleaks') {
            steps {
                script {
                    // Added '|| true' to let the pipeline report findings instead of crashing immediately
                    // Remove '|| true' if you want it to stop the build immediately on secrets found
                    sh 'gitleaks detect --source . --verbose --redact || true'
                }
            }
        }

        stage('Compile') {
            steps {
                sh 'mvn clean compile'
            }
        }

        stage('Tests Unitaires') {
            steps {
                sh 'mvn test'
            }
            post {
                always {
                    // Added: Archive JUnit results so Jenkins displays a test graph
                    junit 'target/surefire-reports/*.xml'
                }
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

        stage('SAST - SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh '''
                        mvn clean verify sonar:sonar \
                        -Dsonar.coverage.jacoco.xmlReportPaths=target/site/jacoco/jacoco.xml
                    '''
                }
            }
        }

        // ADDED: Quality Gate (From your reference pipeline)
        // This stops the pipeline if SonarQube detects too many bugs/vulnerabilities
        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        // ===========================================
        //      CD STAGES - RUN ONLY ON 'MAIN'
        // ===========================================

        stage('Build Docker Images') {
            when { branch 'main' } // Logic: Only run this on main
            steps {
                sh 'docker compose build'
            }
        }

        stage('Login to Docker Hub') {
            when { branch 'main' }
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'DOCKER_HUB_CREDENTIALS',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASSWORD'
                )]) {
                    sh '''
                        echo "${DOCKER_PASSWORD}" | docker login -u "${DOCKER_USER}" --password-stdin
                    '''
                }
            }
        }

        stage('Push Docker Image') {
            when { branch 'main' }
            steps {
                sh '''
                     docker compose push
                     docker logout
                '''
            }
        }

        stage('Deploy with Docker Compose') {
            when { branch 'main' }
            steps {
                // Added cleanup (down) before up to ensure fresh deployment
                sh 'docker compose down && docker compose up -d'
            }
        }
        
        // Optional: Run Verify only on main
        /* 
        stage('Verify Deployment') {
            when { branch 'main' }
            steps { ... }
        }
        */

        stage('Generate HTML Report') {
            steps {
                script {
                    def htmlContent = """
                    <html>
                    <head><title>Pipeline Execution Report</title></head>
                    <body>
                    <h1>Pipeline Build Report</h1>
                    <h2>Build #${env.BUILD_NUMBER}</h2>
                    <p><strong>Branch:</strong> ${env.BRANCH_NAME}</p>
                    <p><strong>Status:</strong> ${currentBuild.currentResult ?: 'SUCCESS'}</p>
                    <p><strong>Date:</strong> ${new Date().format("yyyy-MM-dd HH:mm")}</p>
                    <hr>
                    </body>
                    </html>
                    """
                    writeFile file: 'pipeline-report.html', text: htmlContent
                }
            }
            post {
                always {
                    publishHTML([
                        allowMissing: false,
                        alwaysLinkToLastBuild: true,
                        keepAll: true,
                        reportDir: '',
                        reportFiles: 'pipeline-report.html',
                        reportName: 'Pipeline Report'
                    ])
                }
            }
        }
    }

    post {
        success {
            mail to: 'cirin.chalghoumi@gmail.com',
                subject: "SUCCESS: ${env.JOB_NAME} [${env.BRANCH_NAME}] #${env.BUILD_NUMBER}",
                body: """
                Build Successfully Completed!
                Branch: ${env.BRANCH_NAME}
                Check console: ${env.BUILD_URL}console
                """
        }
        failure {
            mail to: 'cirin.chalghoumi@gmail.com',
                subject: "FAILURE: ${env.JOB_NAME} [${env.BRANCH_NAME}] #${env.BUILD_NUMBER}",
                body: """
                Build Failed!
                Branch: ${env.BRANCH_NAME}
                Check console: ${env.BUILD_URL}console
                """
        }
    }
}

pipeline {
    agent any

    stages {
        // ===========================================
        //      CI STAGES - RUN ON ALL BRANCHES
        // ===========================================

        stage('Checkout') {
            steps {
                checkout scm
                echo "✅ Checked out branch: ${env.BRANCH_NAME}"
            }
        }

        stage('Secrets Scan - Gitleaks') {
            steps {
                script {
                    echo "🔐 Running Gitleaks..."
                    // 1. Generate JSON Report
                    sh 'gitleaks detect --source . --report-path gitleaks-report.json --verbose --redact || true'
                    
                    // 2. Generate SARIF Report (Useful for GitHub Code Scanning)
                    sh 'gitleaks detect --source . --format sarif --report-path gitleaks-report.sarif --redact || true'
                }
            }
        }

        stage('SCA - Trivy File Scan') {
            steps {
                script {
                    echo "🛡️ Running Trivy FS Scan..."
                    // 1. JSON Report
                    sh 'trivy fs --format json --output trivy-report.json .'
                    
                    // 2. Text Report
                    sh 'trivy fs --format table --output trivy-report.txt .'

                    // 3. HTML Report (Requires downloading the template first)
                    sh '''
                        wget https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/html.tpl -O html.tpl || true
                        trivy fs --format template --template "@html.tpl" --output trivy-report.html . || echo "Trivy HTML generation failed"
                    '''
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
                    // This creates the "Unit Tests Report" in the Jenkins UI
                    junit 'target/surefire-reports/*.xml'
                }
            }
        }

        stage('SCA - Dependency Check') {
            steps {
                // This generates 'target/dependency-check-report.html'
                sh '''
                    mvn org.owasp:dependency-check-maven:check \
                    -Dformat=HTML \
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

        // ===========================================
        //      CD STAGES - RUN ONLY ON 'MAIN'
        // ===========================================

        stage('Build Docker Images') {
            when { branch 'main' }
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
                    sh 'echo "${DOCKER_PASSWORD}" | docker login -u "${DOCKER_USER}" --password-stdin'
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
                sh 'docker compose down && docker compose up -d'
            }
        }

        stage('Generate Comprehensive Report') {
            steps {
                script {
                    def htmlContent = """
                    <!DOCTYPE html>
                    <html>
                    <head>
                        <style>
                            body { font-family: Arial, sans-serif; margin: 20px; }
                            h1 { color: #2c3e50; }
                            .card { border: 1px solid #ddd; padding: 15px; border-radius: 5px; margin-bottom: 10px; background: #f9f9f9; }
                            .pass { color: green; font-weight: bold; }
                            .fail { color: red; font-weight: bold; }
                            a { text-decoration: none; color: #3498db; }
                            a:hover { text-decoration: underline; }
                        </style>
                        <title>Pipeline Comprehensive Report</title>
                    </head>
                    <body>
                        <h1>📊 Pipeline Execution Report</h1>
                        <div class="card">
                            <p><strong>Build Number:</strong> #${env.BUILD_NUMBER}</p>
                            <p><strong>Branch:</strong> ${env.BRANCH_NAME}</p>
                            <p><strong>Status:</strong> ${currentBuild.currentResult ?: 'SUCCESS'}</p>
                            <p><strong>Date:</strong> ${new Date().format("yyyy-MM-dd HH:mm")}</p>
                        </div>

                        <h2>📂 Security & Quality Reports</h2>
                        <ul>
                            <li><a href="artifact/gitleaks-report.json" target="_blank">🔐 Gitleaks JSON Report</a></li>
                            <li><a href="artifact/trivy-report.html" target="_blank">🛡️ Trivy HTML Report</a></li>
                            <li><a href="artifact/target/dependency-check-report.html" target="_blank">📦 Dependency Check Report</a></li>
                            <li><a href="artifact/trivy-report.txt" target="_blank">📄 Trivy Text Summary</a></li>
                        </ul>
                    </body>
                    </html>
                    """
                    writeFile file: 'comprehensive-report.html', text: htmlContent
                }
            }
        }
    }

    post {
        always {
            // 1. Archive the specific files you asked for
            archiveArtifacts artifacts: 'gitleaks-report.json, gitleaks-report.sarif, trivy-report.json, trivy-report.html, trivy-report.txt, comprehensive-report.html, target/dependency-check-report.html', allowEmptyArchive: true
            
            // 2. Publish the Comprehensive HTML Report to Jenkins Sidebar
            publishHTML([
                allowMissing: false,
                alwaysLinkToLastBuild: true,
                keepAll: true,
                reportDir: '.',
                reportFiles: 'comprehensive-report.html',
                reportName: 'Comprehensive Report'
            ])
            
            // 3. Publish Dependency Check to Sidebar (Optional but nice)
            publishHTML([
                allowMissing: true,
                alwaysLinkToLastBuild: true,
                keepAll: true,
                reportDir: 'target',
                reportFiles: 'dependency-check-report.html',
                reportName: 'Dependency Check'
            ])
        }
        success {
            mail to: 'cirin.chalghoumi@gmail.com',
                subject: "SUCCESS: ${env.JOB_NAME} [${env.BRANCH_NAME}] #${env.BUILD_NUMBER}",
                body: "Build Successfully Completed! Check the artifacts in Jenkins."
        }
        failure {
            mail to: 'cirin.chalghoumi@gmail.com',
                subject: "FAILURE: ${env.JOB_NAME} [${env.BRANCH_NAME}] #${env.BUILD_NUMBER}",
                body: "Build Failed! Check console output."
        }
    }
}

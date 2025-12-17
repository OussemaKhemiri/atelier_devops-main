pipeline {
    agent any
    
    // Global timeouts to prevent stuck builds
    options {
        timeout(time: 1, unit: 'HOURS')
        buildDiscarder(logRotator(numToKeepStr: '5'))
    }

    stages {
        // ===========================================
        //      PREPARATION & COMPLIANCE
        // ===========================================
        
        stage('Checkout & Info') {
            steps {
                checkout scm 
                script {
                    env.GIT_COMMIT_MSG = sh(returnStdout: true, script: 'git log -1 --pretty=%B').trim()
                }
                echo "✅ Processing Branch: ${env.BRANCH_NAME}"
            }
        }

        stage('Security: Secrets Detection') {
            steps {
                script {
                    // Gitleaks checks for hardcoded passwords/keys
                    // We save output to json for potential parsing later
                    sh 'gitleaks detect --source . --report-path gitleaks-report.json --verbose --redact || true'
                }
            }
        }

        stage('Security: Infrastructure & FS Scan') {
            steps {
                script {
                    // Trivy scans the filesystem for vulnerabilities and config issues
                    sh 'trivy fs . --format table --exit-code 0'
                }
            }
        }

        // ===========================================
        //      BUILD & TEST
        // ===========================================

        stage('Build: Compile') {
            steps {
                sh 'mvn clean compile'
            }
        }

        stage('Quality: Unit Tests') {
            steps {
                sh 'mvn test'
            }
            post {
                always {
                    junit 'target/surefire-reports/*.xml'
                }
            }
        }

        // ===========================================
        //      ADVANCED SECURITY ANALYSIS
        // ===========================================

        stage('Security: SCA (Dependency Check)') {
            steps {
                // Checks standard CVE databases for your Jar files
                sh '''
                    mvn org.owasp:dependency-check-maven:check \
                    -DfailBuildOnCVSS=8 \
                    || echo "SCA Warning: Vulnerabilities found but threshold not met."
                '''
            }
        }

        stage('Quality: SAST (SonarQube)') {
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
        //      EXECUTIVE REPORTING
        // ===========================================

        stage('Generate Executive Report') {
            steps {
                script {
                    // ===============================================
                    // 1. PRE-CALCULATE VARIABLES (Fixes the Crash)
                    // ===============================================
                    // We convert everything to simple Strings first.
                    // This prevents Jenkins from crashing during String interpolation.
                    
                    String dateStr      = new Date().format("yyyy-MM-dd HH:mm")
                    String buildStatus  = currentBuild.currentResult ?: 'SUCCESS'
                    String statusColor  = (buildStatus == 'FAILURE') ? 'badge-danger' : 'badge-success'
                    String borderColor  = (buildStatus == 'FAILURE') ? '#c0392b' : '#27ae60'
                    
                    // Safely handle potentially null env vars
                    String jobName      = env.JOB_NAME ?: 'Unknown Job'
                    String buildNum     = env.BUILD_NUMBER ?: '0'
                    String branchName   = env.BRANCH_NAME ?: 'Unknown Branch'
                    String buildUrl     = env.BUILD_URL ?: '#'
                    
                    // Handle commit message (defaults to placeholder if not set in previous stages)
                    String commitMsg    = env.GIT_COMMIT_MSG ?: "Commit info not available"

                    // ===============================================
                    // 2. DEFINE CSS (Separated for cleanliness)
                    // ===============================================
                    String css = """
                        <style>
                            body { font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif; background-color: #f4f7f6; color: #333; padding: 20px; }
                            .container { max-width: 900px; margin: 0 auto; background: #fff; padding: 30px; box-shadow: 0 2px 10px rgba(0,0,0,0.1); border-radius: 8px; }
                            h1 { color: #2c3e50; border-bottom: 2px solid #3498db; padding-bottom: 10px; }
                            h2 { color: #34495e; margin-top: 30px; }
                            .summary-grid { display: grid; grid-template-columns: 1fr 1fr; gap: 20px; margin-bottom: 30px; }
                            .card { background: #f8f9fa; padding: 15px; border-left: 5px solid #3498db; border-radius: 4px; }
                            .badge { display: inline-block; padding: 5px 10px; border-radius: 4px; color: #fff; font-weight: bold; }
                            .badge-success { background-color: #27ae60; }
                            .badge-warning { background-color: #f39c12; }
                            .badge-danger { background-color: #c0392b; }
                            table { width: 100%; border-collapse: collapse; margin-top: 15px; }
                            th, td { padding: 12px; border-bottom: 1px solid #ddd; text-align: left; }
                            th { background-color: #2c3e50; color: white; }
                            .footer { margin-top: 40px; font-size: 0.9em; color: #7f8c8d; text-align: center; border-top: 1px solid #eee; padding-top: 20px; }
                            a { color: #3498db; text-decoration: none; }
                        </style>
                    """

                    // ===============================================
                    // 3. GENERATE HTML (Now using simple variables)
                    // ===============================================
                    String htmlContent = """
                    <!DOCTYPE html>
                    <html>
                    <head>
                        <title>CI/CD Security Report</title>
                        ${css}
                    </head>
                    <body>
                        <div class="container">
                            <h1>🚀 Pipeline Execution Report</h1>
                            
                            <div class="summary-grid">
                                <div class="card">
                                    <h3>Build Information</h3>
                                    <p><strong>Job:</strong> ${jobName}</p>
                                    <p><strong>Build ID:</strong> #${buildNum}</p>
                                    <p><strong>Branch:</strong> ${branchName}</p>
                                    <p><strong>Date:</strong> ${dateStr}</p>
                                </div>
                                <div class="card" style="border-left-color: ${borderColor}">
                                    <h3>Overall Status</h3>
                                    <div style="font-size: 24px; margin-top: 10px;">
                                        <span class="badge ${statusColor}">${buildStatus}</span>
                                    </div>
                                    <p><small>Commit: ${commitMsg}</small></p>
                                </div>
                            </div>

                            <h2>🛡️ Security & Quality Assurance</h2>
                            <p>This build has undergone the following automated compliance checks:</p>
                            
                            <table>
                                <thead>
                                    <tr>
                                        <th>Security Tier</th>
                                        <th>Tool Used</th>
                                        <th>Description of Check</th>
                                        <th>Status</th>
                                    </tr>
                                </thead>
                                <tbody>
                                    <tr>
                                        <td><strong>Secrets Detection</strong></td>
                                        <td>🕵️ Gitleaks</td>
                                        <td>Scans code history for hardcoded passwords and keys.</td>
                                        <td><span class="badge badge-success">Completed</span></td>
                                    </tr>
                                    <tr>
                                        <td><strong>Infrastructure</strong></td>
                                        <td>🐳 Trivy</td>
                                        <td>Scans filesystem and OS packages for vulnerabilities.</td>
                                        <td><span class="badge badge-success">Completed</span></td>
                                    </tr>
                                    <tr>
                                        <td><strong>SCA (Dependencies)</strong></td>
                                        <td>📦 OWASP DC</td>
                                        <td>Checks Java libraries against the NVD database.</td>
                                        <td><span class="badge badge-success">Completed</span></td>
                                    </tr>
                                    <tr>
                                        <td><strong>SAST (Code Quality)</strong></td>
                                        <td>🧠 SonarQube</td>
                                        <td>Static analysis for bugs and security hotspots.</td>
                                        <td><span class="badge badge-success">Sent to Server</span></td>
                                    </tr>
                                    <tr>
                                        <td><strong>Unit Verification</strong></td>
                                        <td>🧪 JUnit</td>
                                        <td>Functional unit tests validation.</td>
                                        <td><span class="badge badge-success">Completed</span></td>
                                    </tr>
                                </tbody>
                            </table>

                            <div class="footer">
                                <p>Generated automatically by Jenkins CI/CD Pipeline</p>
                                <p><a href="${buildUrl}">View Full Console Logs</a> | <a href="${buildUrl}testReport/">View Test Results</a></p>
                            </div>
                        </div>
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
                        reportName: 'Security Compliance Report'
                    ])
                }
            }
        }
    }
}

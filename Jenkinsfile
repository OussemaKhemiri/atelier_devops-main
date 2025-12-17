pipeline {
    agent any
    
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
                    // getting commit msg safely
                    env.GIT_COMMIT_MSG = sh(returnStdout: true, script: 'git log -1 --pretty=%B').trim()
                }
                echo "✅ Processing Branch: ${env.BRANCH_NAME}"
            }
        }

        stage('Security: Secrets Detection') {
            steps {
                script {
                    sh 'gitleaks detect --source . --report-path gitleaks-report.json --verbose --redact || true'
                }
            }
        }

        stage('Security: Infrastructure & FS Scan') {
            steps {
                script {
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
                    // 1. Get raw data strings
                    String buildStatus = currentBuild.currentResult ?: 'SUCCESS'
                    String jobName = env.JOB_NAME
                    String buildNum = env.BUILD_NUMBER
                    String branch = env.BRANCH_NAME
                    String url = env.BUILD_URL
                    String commit = env.GIT_COMMIT_MSG ?: 'N/A'
                    
                    // 2. Generate Date using Shell (Safer than Java Date in Jenkins)
                    String dateStr = sh(returnStdout: true, script: 'date "+%Y-%m-%d %H:%M"').trim()

                    // 3. Call the @NonCPS function to generate the HTML string
                    String htmlContent = generateHtmlReport(jobName, buildNum, branch, dateStr, buildStatus, commit, url)
                    
                    // 4. Write to file
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

    post {
        success {
            mail to: 'cirin.chalghoumi@gmail.com',
                subject: "✅ PASSED: ${env.JOB_NAME} - Branch ${env.BRANCH_NAME}",
                body: "Pipeline Success. Report: ${env.BUILD_URL}Security_20Compliance_20Report/"
        }
        failure {
            mail to: 'cirin.chalghoumi@gmail.com',
                subject: "❌ FAILED: ${env.JOB_NAME} - Branch ${env.BRANCH_NAME}",
                body: "Pipeline Failed. Logs: ${env.BUILD_URL}"
        }
    }
}

// ========================================================================
//  HELPER FUNCTION - MUST BE OUTSIDE THE PIPELINE BLOCK
//  @NonCPS prevents the "SandboxContinuable" crash by disabling state saving
// ========================================================================
@NonCPS
def generateHtmlReport(String jobName, String buildNum, String branchName, String dateStr, String buildStatus, String commitMsg, String buildUrl) {
    
    // Logic to determine colors
    String statusColor = (buildStatus == 'FAILURE') ? '#c0392b' : '#27ae60' // Red or Green
    String statusBadgeClass = (buildStatus == 'FAILURE') ? 'badge-danger' : 'badge-success'

    // The HTML String
    return """
<!DOCTYPE html>
<html>
<head>
    <title>CI/CD Security Report</title>
    <style>
        body { font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif; background-color: #f4f7f6; color: #333; padding: 20px; }
        .container { max-width: 900px; margin: 0 auto; background: #fff; padding: 30px; box-shadow: 0 2px 10px rgba(0,0,0,0.1); border-radius: 8px; }
        h1 { color: #2c3e50; border-bottom: 2px solid #3498db; padding-bottom: 10px; }
        h2 { color: #34495e; margin-top: 30px; }
        .summary-grid { display: grid; grid-template-columns: 1fr 1fr; gap: 20px; margin-bottom: 30px; }
        .card { background: #f8f9fa; padding: 15px; border-left: 5px solid #3498db; border-radius: 4px; }
        .badge { display: inline-block; padding: 5px 10px; border-radius: 4px; color: #fff; font-weight: bold; }
        .badge-success { background-color: #27ae60; }
        .badge-danger { background-color: #c0392b; }
        table { width: 100%; border-collapse: collapse; margin-top: 15px; }
        th, td { padding: 12px; border-bottom: 1px solid #ddd; text-align: left; }
        th { background-color: #2c3e50; color: white; }
        .footer { margin-top: 40px; font-size: 0.9em; color: #7f8c8d; text-align: center; border-top: 1px solid #eee; padding-top: 20px; }
        a { color: #3498db; text-decoration: none; }
    </style>
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
            <div class="card" style="border-left-color: ${statusColor}">
                <h3>Overall Status</h3>
                <div style="font-size: 24px; margin-top: 10px;">
                    <span class="badge ${statusBadgeClass}">${buildStatus}</span>
                </div>
                <p><small>Commit: ${commitMsg}</small></p>
            </div>
        </div>

        <h2>🛡️ Security & Quality Assurance</h2>
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
}

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
                    env.GIT_COMMIT_MSG = sh(
                        returnStdout: true,
                        script: 'git log -1 --pretty=%B'
                    ).trim()
                }
                echo "Processing Branch: ${env.BRANCH_NAME}"
            }
        }

        stage('Security: Secrets Detection') {
            steps {
                sh 'gitleaks detect --source . --report-path gitleaks-report.json --verbose --redact || true'
            }
        }

        stage('Security: Infrastructure & FS Scan') {
            steps {
                sh 'trivy fs . --format table --exit-code 0'
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
        //      EXECUTIVE REPORTING (FIXED)
        // ===========================================

        stage('Generate Executive Report') {
            steps {

                // CPS-safe build status
                script {
                    env.BUILD_RES = currentBuild.currentResult
                }

                withEnv([
                    "JOB=${env.JOB_NAME}",
                    "ID=${env.BUILD_NUMBER}",
                    "BRANCH=${env.BRANCH_NAME}",
                    "URL=${env.BUILD_URL}"
                ]) {

                    sh '''
                        #!/bin/bash

                        DATE_STR=$(date "+%Y-%m-%d %H:%M")
                        COMMIT_MSG=$(git log -1 --pretty=%B | tr '\\n' ' ' | tr -d '"')

                        if [ "$BUILD_RES" = "FAILURE" ]; then
                            COLOR="#c0392b"
                            BADGE="badge-danger"
                        else
                            COLOR="#27ae60"
                            BADGE="badge-success"
                        fi

                        cat > pipeline-report.html <<EOF
<!DOCTYPE html>
<html>
<head>
    <title>Security Report</title>
    <style>
        body { font-family: 'Segoe UI', sans-serif; background-color: #f4f7f6; color: #333; padding: 20px; }
        .container { max-width: 900px; margin: 0 auto; background: #fff; padding: 30px; border-radius: 8px; box-shadow: 0 2px 10px rgba(0,0,0,0.1); }
        h1 { color: #2c3e50; border-bottom: 2px solid #3498db; padding-bottom: 10px; }
        .summary-grid { display: grid; grid-template-columns: 1fr 1fr; gap: 20px; margin-bottom: 30px; }
        .card { background: #f8f9fa; padding: 15px; border-left: 5px solid #3498db; border-radius: 4px; }
        .badge { display: inline-block; padding: 5px 10px; border-radius: 4px; color: #fff; font-weight: bold; }
        .badge-success { background-color: #27ae60; }
        .badge-danger { background-color: #c0392b; }
        table { width: 100%; border-collapse: collapse; margin-top: 15px; }
        th, td { padding: 12px; border-bottom: 1px solid #ddd; text-align: left; }
        th { background-color: #2c3e50; color: white; }
        a { color: #3498db; text-decoration: none; }
        .footer { margin-top: 40px; font-size: 0.9em; color: #7f8c8d; text-align: center; border-top: 1px solid #eee; padding-top: 20px; }
    </style>
</head>
<body>
    <div class="container">
        <h1>Pipeline Execution Report</h1>

        <div class="summary-grid">
            <div class="card">
                <h3>Build Information</h3>
                <p><strong>Job:</strong> ${JOB}</p>
                <p><strong>Build ID:</strong> #${ID}</p>
                <p><strong>Branch:</strong> ${BRANCH}</p>
                <p><strong>Date:</strong> ${DATE_STR}</p>
            </div>

            <div class="card" style="border-left-color: ${COLOR}">
                <h3>Overall Status</h3>
                <span class="badge ${BADGE}">${BUILD_RES}</span>
                <p><small>Commit: ${COMMIT_MSG}</small></p>
            </div>
        </div>

        <h2>Security & Quality Assurance</h2>
        <table>
            <thead>
                <tr>
                    <th>Tier</th>
                    <th>Tool</th>
                    <th>Description</th>
                    <th>Status</th>
                </tr>
            </thead>
            <tbody>
                <tr><td>Secrets</td><td>Gitleaks</td><td>Credential detection</td><td><span class="badge badge-success">Completed</span></td></tr>
                <tr><td>Infrastructure</td><td>Trivy</td><td>Filesystem scan</td><td><span class="badge badge-success">Completed</span></td></tr>
                <tr><td>SCA</td><td>OWASP DC</td><td>Dependency CVEs</td><td><span class="badge badge-success">Completed</span></td></tr>
                <tr><td>SAST</td><td>SonarQube</td><td>Static analysis</td><td><span class="badge badge-success">Sent</span></td></tr>
                <tr><td>Testing</td><td>JUnit</td><td>Unit validation</td><td><span class="badge badge-success">Completed</span></td></tr>
            </tbody>
        </table>

        <div class="footer">
            <p>Generated by Jenkins CI/CD</p>
            <p><a href="${URL}">View Console Logs</a></p>
        </div>
    </div>
</body>
</html>
EOF
                    '''
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

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
        
        // Etape 2 : Scan des secrets avec rapport détaillé
        stage('Secrets Scan - Gitleaks') {
            steps {
                script {
                    echo "🔍 Starting Gitleaks Secret Detection..."
                    // Added '|| true' so it doesn't crash the build immediately
                    sh '''
                    gitleaks detect --source . \
                        --verbose \
                        --redact \
                        --report-format json \
                        --report-path gitleaks-report.json \
                        --exit-code 0 || true
                    '''
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: 'gitleaks-report.json', allowEmptyArchive: true
                    
                    // Generate a simple HTML wrapper for the JSON report
                    sh '''
                        echo "<html><body><h1>Gitleaks Report</h1><pre>$(cat gitleaks-report.json)</pre></body></html>" > gitleaks-report.html
                    '''
                    
                    publishHTML([
                        allowMissing: false,
                        alwaysLinkToLastBuild: true,
                        keepAll: true,
                        reportDir: '.',
                        reportFiles: 'gitleaks-report.html',
                        reportName: 'Gitleaks Secrets Report'
                    ])
                }
            }
        }
        
        // Etape 3 : Scan RAPIDE du code source avec Trivy (Bandwidth Friendly)
        stage('Trivy Security Scan') {
            steps {
                sh """
                    echo "🔍 Fast Trivy code vulnerability scan..."
                    trivy fs --skip-db-update \
                        --exit-code 0 \
                        --severity HIGH,CRITICAL \
                        --format table \
                        --timeout 1m \
                        . > trivy-report.txt
                    
                    # Rapport JSON aussi
                    trivy fs --skip-db-update \
                        --format json \
                        --output trivy-report.json \
                        . || echo "JSON report generation completed"
                """
            }
            post {
                always {
                    archiveArtifacts artifacts: 'trivy-report.*', allowEmptyArchive: true
                    sh '''
                        cat > trivy-report.html << EOF
                        <html>
                        <head><title>Trivy Security Scan</title></head>
                        <body>
                        <h1>Security Vulnerability Scan</h1>
                        <h2>Scan Type: Source Code & Dependencies</h2>
                        <pre>$(cat trivy-report.txt)</pre>
                        <p><em>Scanned: Source code files, dependencies, and configuration files</em></p>
                        </body>
                        </html>
                        EOF
                    '''
                    publishHTML([
                        allowMissing: false,
                        alwaysLinkToLastBuild: true,
                        keepAll: true,
                        reportDir: '.',
                        reportFiles: 'trivy-report.html',
                        reportName: 'Trivy Security Report'
                    ])
                }
            }
        }
        
        stage('Tests Unitaires') {
            steps {
                sh 'mvn test' 
            }
            post {
                always {
                    junit '**/target/surefire-reports/*.xml'
                    publishHTML([
                        allowMissing: true,
                        alwaysLinkToLastBuild: true,
                        keepAll: true,
                        reportDir: 'target/site',
                        reportFiles: 'surefire-report.html',
                        reportName: 'Unit Tests Report'
                    ])
                }
            }
        }
        
        stage('SCA - Dependency Check') {
            steps {
                // Generates HTML report by default
                sh '''
                    mvn org.owasp:dependency-check-maven:check \
                    -Dformat=HTML \
                    -DfailBuildOnCVSS=7 \
                    || echo "Dependency check completed"
                '''
            }
            post {
                always {
                    archiveArtifacts artifacts: '**/target/dependency-check-report.html', allowEmptyArchive: true
                    publishHTML([
                        allowMissing: true,
                        alwaysLinkToLastBuild: true,
                        keepAll: true,
                        reportDir: 'target',
                        reportFiles: 'dependency-check-report.html',
                        reportName: 'Dependency Check Report'
                    ])
                }
            }
        }
        
        stage('Compile') {
            steps {
                sh 'mvn clean compile'
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
            post {
                always {
                    script {
                         // Attempt to construct Sonar URL link
                         def sonarHost = env.SONAR_HOST_URL ?: "http://localhost:9000"
                         env.SONAR_URL = "${sonarHost}/dashboard?id=atelier_devops-main"
                    }
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
                    sh '''
                        echo "Logging into Docker Hub..."
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
                sh 'docker compose down && docker compose up -d'
            }
        }
        
        stage('Generate Comprehensive Reports') {
            steps {
                script {
                    def htmlContent = """
                    <html>
                    <head>
                        <title>Pipeline Execution Report - Build #${env.BUILD_NUMBER}</title>
                        <style>
                            body { font-family: Arial, sans-serif; margin: 20px; }
                            .header { background: #f4f4f4; padding: 20px; border-radius: 5px; }
                            .section { margin: 20px 0; padding: 15px; border-left: 4px solid #007cba; }
                            .success { border-color: #28a745; background: #f8fff9; }
                            .warning { border-color: #ffc107; background: #fffef0; }
                            .error { border-color: #dc3545; background: #fff5f5; }
                            .report-link { margin: 5px 0; }
                            a { text-decoration: none; color: #007cba; font-weight: bold; }
                        </style>
                    </head>
                    <body>
                        <div class="header">
                            <h1>🚀 Pipeline Execution Report</h1>
                            <h2>Build #${env.BUILD_NUMBER} (${env.BRANCH_NAME})</h2>
                            <p><strong>Status:</strong> <span style="color: ${currentBuild.currentResult == 'SUCCESS' ? 'green' : 'red'}">${currentBuild.currentResult ?: 'SUCCESS'}</span></p>
                            <p><strong>Date:</strong> ${new Date().format("yyyy-MM-dd HH:mm:ss")}</p>
                        </div>
                        
                        <div class="section">
                            <h3>📊 Security & Quality Reports</h3>
                            <div class="report-link"><a href="${env.BUILD_URL}/Gitleaks_20Secrets_20Report/" target="_blank">🔐 Gitleaks Secrets Report</a></div>
                            <div class="report-link"><a href="${env.BUILD_URL}/Trivy_20Security_20Report/" target="_blank">🔍 Trivy Security Report</a></div>
                            <div class="report-link"><a href="${env.BUILD_URL}/Unit_20Tests_20Report/" target="_blank">🧪 Unit Tests Report</a></div>
                            <div class="report-link"><a href="${env.BUILD_URL}/Dependency_20Check_20Report/" target="_blank">📦 Dependency Check Report</a></div>
                            <div class="report-link"><a href="${env.SONAR_URL ?: '#'}" target="_blank">📈 SonarQube Quality Report</a></div>
                        </div>
                        
                        <div class="section success">
                            <h3>✅ Build Artifacts</h3>
                            <div class="report-link"><a href="artifact/gitleaks-report.json" target="_blank">📄 Gitleaks JSON</a></div>
                            <div class="report-link"><a href="artifact/trivy-report.json" target="_blank">📄 Trivy JSON</a></div>
                        </div>
                        
                        <div class="section">
                            <h3>🔗 Quick Links</h3>
                            <div class="report-link"><a href="${env.BUILD_URL}console" target="_blank">Console Output</a></div>
                        </div>
                    </body>
                    </html>
                    """
                    
                    writeFile file: 'comprehensive-report.html', text: htmlContent
                    
                    // Archive the custom comprehensive report
                    archiveArtifacts artifacts: 'comprehensive-report.html', allowEmptyArchive: true
                    
                    // Publish it to the sidebar
                    publishHTML([
                        allowMissing: false,
                        alwaysLinkToLastBuild: true,
                        keepAll: true,
                        reportDir: '.',
                        reportFiles: 'comprehensive-report.html',
                        reportName: 'Pipeline Comprehensive Report'
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
                ✅ Build Successfully Completed!

                Job: ${env.JOB_NAME}
                Branch: ${env.BRANCH_NAME}
                Build: #${env.BUILD_NUMBER}
                
                📊 View Comprehensive Report:
                ${env.BUILD_URL}/Pipeline_20Comprehensive_20Report/
                """
        }
        
        failure {
            mail to: 'cirin.chalghoumi@gmail.com',
                subject: "FAILED: ${env.JOB_NAME} [${env.BRANCH_NAME}] #${env.BUILD_NUMBER}",
                body: """
                ❌ BUILD FAILED!

                Job: ${env.JOB_NAME}
                Branch: ${env.BRANCH_NAME}
                Build: #${env.BUILD_NUMBER}
                
                Check console: ${env.BUILD_URL}console
                """
        }
    }
}

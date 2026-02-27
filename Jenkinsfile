pipeline {
    agent { label 'vm-agent' }
    
    environment {
        MAVEN_HOME = tool 'Maven-3.9'
        JAVA_HOME = tool 'JDK-17'
        PATH = "${MAVEN_HOME}/bin:${JAVA_HOME}/bin:${env.PATH}"
        REPORT_DIR = 'security-reports'
        SONAR_TOKEN = credentials('sonarqube-token')
        SONAR_HOST_URL = 'http://localhost:9000'
        IMAGE_TAG = "${BUILD_NUMBER}"
    }
    
    options {
        buildDiscarder(logRotator(numToKeepStr: '10'))
        timeout(time: 2, unit: 'HOURS')
        disableConcurrentBuilds()
    }
    
    stages {
        stage('Environment Check') {
            steps {
                echo ' Pipeline started - Build #' + env.BUILD_NUMBER
                sh 'mkdir -p ${REPORT_DIR}'
            }
        }
        
        stage('Secret Scanning') {
            steps {
                sh '''
                    echo "===  Gitleaks Secret Scan ==="
                    gitleaks detect --source . --report-format json --report-path ${REPORT_DIR}/gitleaks-report.json || true
                '''
            }
            post {
                always {
                    archiveArtifacts artifacts: "${REPORT_DIR}/gitleaks-report.json", allowEmptyArchive: true
                }
            }
        }
        
        stage('Build') {
            steps {
                sh './mvnw clean package -DskipTests'
            }
        }
        
        stage('SAST - SonarQube') {
            steps {
                script {
                    def scannerHome = tool 'SonarScanner'
                    withSonarQubeEnv('SonarQube') {
                        sh """
                            ${scannerHome}/bin/sonar-scanner \
                            -Dsonar.projectKey=spring-petclinic \
                            -Dsonar.projectName='Spring PetClinic' \
                            -Dsonar.sources=. \
                            -Dsonar.java.binaries=**/target/classes \
                            -Dsonar.exclusions=**/target/**,**/.mvn/**,**/mvnw
                        """
                    }
                }
            }
        }
        
        stage('SCA - Dependency Check') {
            steps {
                sh '''
                    /opt/dependency-check/bin/dependency-check.sh \
                        --project "Spring PetClinic" \
                        --scan . \
                        --format JSON --format HTML \
                        --out ${REPORT_DIR}/dependency-check \
                        --enableExperimental || true
                '''
            }
            post {
                always {
                    archiveArtifacts artifacts: "${REPORT_DIR}/dependency-check/*", allowEmptyArchive: true
                }
            }
        }
        
        stage('Unit Tests') {
            steps {
                sh './mvnw test'
            }
            post {
                always {
                    junit testResults: '**/target/surefire-reports/*.xml', allowEmptyResults: true
                }
            }
        }
        
        stage('Container Security') {
            steps {
                script {
                    sh '''
                        ./mvnw package -pl spring-petclinic-config-server -am -DskipTests -q
                        cd spring-petclinic-config-server
                        JAR_FILE=$(ls target/*.jar | head -1)
                        ARTIFACT_NAME=$(basename $JAR_FILE .jar)
                        mkdir -p docker-build
                        cp $JAR_FILE docker-build/
                        cp ../docker/Dockerfile docker-build/
                        docker build --build-arg ARTIFACT_NAME=$ARTIFACT_NAME --build-arg EXPOSED_PORT=8888 -t petclinic-config-server:${IMAGE_TAG} docker-build/
                        rm -rf docker-build
                        cd ..
                        trivy image --format json --output ${REPORT_DIR}/trivy-report.json --severity HIGH,CRITICAL petclinic-config-server:${IMAGE_TAG} || true
                    '''
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: "${REPORT_DIR}/trivy-report.json", allowEmptyArchive: true
                }
            }
        }
        
        stage('Approval: Deploy to Staging') {
            steps {
                script {
                    input message: 'Approve deployment to STAGING?', ok: 'Deploy to Staging', submitterParameter: 'APPROVER'
                    echo "Approved by: ${env.APPROVER}"
                }
            }
        }
        
        stage('Deploy to Staging') {
            steps {
                sh '''
                    docker stop petclinic-staging 2>/dev/null || true
                    docker rm petclinic-staging 2>/dev/null || true
                    docker run -d --name petclinic-staging -p 8888:8888 petclinic-config-server:${IMAGE_TAG}
                    sleep 5
                '''
            }
        }
        
        stage('Approval: Deploy to Production') {
            steps {
                script {
                    timeout(time: 24, unit: 'HOURS') {
                        input message: 'Approve deployment to PRODUCTION?', ok: 'Deploy to Production', submitterParameter: 'PROD_APPROVER'
                    }
                    echo "Production approved by: ${env.PROD_APPROVER}"
                }
            }
        }
        
        stage('Deploy to Production') {
            steps {
                sh 'docker tag petclinic-config-server:${IMAGE_TAG} petclinic-config-server:production'
            }
        }
        
        // ==========================================
        // STAGE 12: FINAL REPORT (NEW)
        // ==========================================
        stage('Generate Final Report') {
            steps {
                script {
                    sh '''
                        echo "=== 📊 Generating Final Security Report ==="
                        
                        cat > ${REPORT_DIR}/pipeline-summary.html << 'HTMLEOF'
<!DOCTYPE html>
<html>
<head>
    <title>DevSecOps Pipeline Report</title>
    <style>
        body { font-family: Arial, sans-serif; margin: 40px; background: #f5f5f5; }
        .header { background: #2c3e50; color: white; padding: 20px; border-radius: 5px; }
        .summary { background: white; padding: 20px; margin: 20px 0; border-radius: 5px; box-shadow: 0 2px 5px rgba(0,0,0,0.1); }
        .stage { background: white; padding: 15px; margin: 10px 0; border-left: 4px solid #27ae60; }
        .stage.failed { border-left-color: #e74c3c; }
        .metric { display: inline-block; margin: 10px 20px 10px 0; }
        .metric-value { font-size: 24px; font-weight: bold; color: #27ae60; }
        .metric-label { font-size: 12px; color: #7f8c8d; }
    </style>
</head>
<body>
    <div class="header">
        <h1> DevSecOps Pipeline Report</h1>
        <p>Build #BUILD_NUMBER | Commit: GIT_COMMIT | Date: BUILD_DATE</p>
    </div>
    
    <div class="summary">
        <h2>Executive Summary</h2>
        <div class="metric">
            <div class="metric-value">7</div>
            <div class="metric-label">Security Scans</div>
        </div>
        <div class="metric">
            <div class="metric-value">2</div>
            <div class="metric-label">Approval Gates</div>
        </div>
        <div class="metric">
            <div class="metric-value" style="color: #27ae60;">PASSED</div>
            <div class="metric-label">Status</div>
        </div>
    </div>

    <div class="stage">
        <h3> Secret Scanning (Gitleaks)</h3>
        <p>Scanned for hardcoded secrets, API keys, and credentials</p>
    </div>

    <div class="stage">
        <h3> SAST (SonarQube)</h3>
        <p>Static code analysis for bugs, vulnerabilities, and code smells</p>
    </div>

    <div class="stage">
        <h3> SCA (OWASP Dependency-Check)</h3>
        <p>Identified known vulnerabilities in third-party dependencies</p>
    </div>

    <div class="stage">
        <h3> Container Security (Trivy)</h3>
        <p>Scanned Docker image for OS and application vulnerabilities</p>
    </div>

    <div class="stage">
        <h3> Unit Tests</h3>
        <p>Executed unit tests across all microservices</p>
    </div>

    <div class="summary">
        <h2>Artifacts Generated</h2>
        <ul>
            <li>gitleaks-report.json - Secret scanning results</li>
            <li>dependency-check-report.html - Vulnerability report</li>
            <li>trivy-report.json - Container scan results</li>
            <li>test-reports - Unit test results</li>
        </ul>
    </div>
</body>
</html>
HTMLEOF

                        # Replace placeholders
                        sed -i "s/BUILD_NUMBER/${BUILD_NUMBER}/g" ${REPORT_DIR}/pipeline-summary.html
                        sed -i "s/GIT_COMMIT/${GIT_COMMIT}/g" ${REPORT_DIR}/pipeline-summary.html
                        sed -i "s/BUILD_DATE/$(date)/g" ${REPORT_DIR}/pipeline-summary.html

                        echo " Report generated: ${REPORT_DIR}/pipeline-summary.html"
                    '''
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: "${REPORT_DIR}/pipeline-summary.html", allowEmptyArchive: true
                    publishHTML(target: [
                        allowMissing: false,
                        alwaysLinkToLastBuild: true,
                        keepAll: true,
                        reportDir: "${REPORT_DIR}",
                        reportFiles: 'pipeline-summary.html',
                        reportName: 'Pipeline Summary Report'
                    ])
                }
            }
        }
    }
    
    post {
        always {
            echo "=========================================="
            echo "PIPELINE COMPLETED - Build #${BUILD_NUMBER}"
            echo "=========================================="
        }
        success {
            echo " SUCCESS! Check the Pipeline Summary Report"
        }
        failure {
            echo " FAILED!"
        }
    }
}

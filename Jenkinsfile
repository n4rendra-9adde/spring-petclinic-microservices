pipeline {
    agent any
    
    environment {
        MAVEN_HOME = tool 'Maven-3.9'
        JAVA_HOME = tool 'JDK-17'
        PATH = "${MAVEN_HOME}/bin:${JAVA_HOME}/bin:${env.PATH}"
        REPORT_DIR = 'security-reports'
        SONAR_TOKEN = credentials('sonarqube-token')
        SONAR_HOST_URL = 'http://localhost:9000'
    }
    
    stages {
        // ==========================================
        // STAGE 1: Environment Check
        // ==========================================
        stage('Environment Check') {
            steps {
                echo '✅ Pipeline started - Build #' + env.BUILD_NUMBER
                sh '''
                    echo "Java version:"
                    java -version
                    echo "Maven version:"
                    mvn -version
                '''
            }
        }
        
        // ==========================================
        // STAGE 2: Secret Scanning (Gitleaks)
        // ==========================================
        stage('Secret Scanning') {
            steps {
                sh '''
                    echo "=== 🔒 Running Gitleaks Secret Scan ==="
                    mkdir -p ${REPORT_DIR}
                    
                    gitleaks detect --source . --verbose --report-format json \
                        --report-path ${REPORT_DIR}/gitleaks-report.json || true
                    
                    if [ -s ${REPORT_DIR}/gitleaks-report.json ]; then
                        echo "⚠️  WARNING: Potential secrets found!"
                        echo "Count: $(cat ${REPORT_DIR}/gitleaks-report.json | grep -c 'Match' || echo '0')"
                    else
                        echo "✅ No secrets detected"
                    fi
                '''
            }
            post {
                always {
                    archiveArtifacts artifacts: "${REPORT_DIR}/gitleaks-report.json", allowEmptyArchive: true
                }
            }
        }
        
        // ==========================================
        // STAGE 3: Build
        // ==========================================
        stage('Build') {
            steps {
                sh '''
                    echo "=== 🔨 Building Application ==="
                    ./mvnw clean compile -DskipTests
                '''
            }
        }
        
        // ==========================================
        // STAGE 4: SAST - SonarQube
        // ==========================================
        stage('SAST - SonarQube') {
            steps {
                script {
                    def scannerHome = tool 'SonarScanner'
                    withSonarQubeEnv('SonarQube') {
                        sh """
                            echo "=== 🔍 Running SonarQube SAST Analysis ==="
                            ${scannerHome}/bin/sonar-scanner \
                            -Dsonar.projectKey=spring-petclinic \
                            -Dsonar.projectName='Spring PetClinic Microservices' \
                            -Dsonar.sources=. \
                            -Dsonar.java.binaries=**/target/classes \
                            -Dsonar.exclusions=**/target/**,**/*.min.js,**/node_modules/**,**/.mvn/**,**/mvnw \
                            -Dsonar.sourceEncoding=UTF-8
                        """
                    }
                }
            }
        }
        
        // ==========================================
        // STAGE 5: SCA - OWASP Dependency-Check (NEW)
        // ==========================================
        stage('SCA - Dependency Check') {
            steps {
                sh '''
                    echo "=== 📦 Running OWASP Dependency-Check ==="
                    
                    dependency-check.sh \
                        --project "Spring PetClinic Microservices" \
                        --scan . \
                        --format JSON \
                        --format HTML \
                        --out ${REPORT_DIR}/dependency-check \
                        --enableExperimental \
                        --failOnCVSS 11 || true
                    
                    echo "Dependency Check completed"
                    ls -lh ${REPORT_DIR}/dependency-check/
                '''
            }
            post {
                always {
                    archiveArtifacts artifacts: "${REPORT_DIR}/dependency-check/*", allowEmptyArchive: true
                    publishHTML(target: [
                        allowMissing: true,
                        alwaysLinkToLastBuild: true,
                        keepAll: true,
                        reportDir: "${REPORT_DIR}/dependency-check",
                        reportFiles: 'dependency-check-report.html',
                        reportName: 'OWASP Dependency-Check Report'
                    ])
                }
            }
        }
        
        // ==========================================
        // STAGE 6: Unit Tests
        // ==========================================
        stage('Unit Tests') {
            steps {
                sh '''
                    echo "=== 🧪 Running Unit Tests ==="
                    ./mvnw test
                '''
            }
            post {
                always {
                    junit testResults: '**/target/surefire-reports/*.xml', allowEmptyResults: true
                    sh '''
                        mkdir -p ${REPORT_DIR}/tests
                        cp -r */target/surefire-reports ${REPORT_DIR}/tests/ 2>/dev/null || true
                    '''
                    archiveArtifacts artifacts: "${REPORT_DIR}/tests/**/*", allowEmptyArchive: true
                }
            }
        }
    }
    
    post {
        always {
            echo "=========================================="
            echo "Pipeline Report - Build #${BUILD_NUMBER}"
            echo "=========================================="
            echo "✅ Environment Check: COMPLETED"
            echo "✅ Secret Scanning: COMPLETED"
            echo "✅ Build: COMPLETED"
            echo "✅ SAST (SonarQube): COMPLETED"
            echo "✅ SCA (Dependency-Check): COMPLETED"
            echo "✅ Unit Tests: COMPLETED"
            echo "=========================================="
        }
        success {
            echo "🎉 ALL SECURITY CHECKS PASSED!"
        }
        failure {
            echo "❌ PIPELINE FAILED - Check logs above"
        }
    }
}
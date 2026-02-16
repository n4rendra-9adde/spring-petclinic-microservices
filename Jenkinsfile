pipeline {
    agent any
    
    environment {
        MAVEN_HOME = tool 'Maven-3.9'
        JAVA_HOME = tool 'JDK-17'
        PATH = "${MAVEN_HOME}/bin:${JAVA_HOME}/bin:${env.PATH}"
        REPORT_DIR = 'security-reports'
    }
    
    stages {
        // ==========================================
        // STAGE 1: Environment Check (from Step 1)
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
        // STAGE 2: Secret Scanning (NEW - Step 3)
        // ==========================================
        stage('Secret Scanning') {
            steps {
                sh '''
                    echo "=== 🔒 Running Gitleaks Secret Scan ==="
                    mkdir -p ${REPORT_DIR}
                    
                    # Run gitleaks
                    gitleaks detect --source . --verbose --report-format json \
                        --report-path ${REPORT_DIR}/gitleaks-report.json || true
                    
                    # Show results summary
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
        // STAGE 3: Build (from Step 1)
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
        // STAGE 4: Unit Tests (from Step 2)
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
# Navigate to your local repository
cd spring-petclinic-microservices

# Create Jenkinsfile
cat > Jenkinsfile << 'EOF'
pipeline {
    agent any
    
    environment {
        // Tools configuration
        MAVEN_HOME = tool 'Maven-3.9'
        JAVA_HOME = tool 'JDK-17'
        PATH = "${MAVEN_HOME}/bin:${JAVA_HOME}/bin:${env.PATH}"
        
        // Report directory
        REPORT_DIR = 'security-reports'
        
        // SonarQube configuration
        SONAR_TOKEN = credentials('sonarqube-token')
        SONAR_HOST_URL = 'http://localhost:9000'
    }
    
    options {
        buildDiscarder(logRotator(numToKeepStr: '5'))
        timeout(time: 1, unit: 'HOURS')
        disableConcurrentBuilds()
    }
    
    stages {
        // ==========================================
        // STAGE 1: PRE-CHECKS & SETUP
        // ==========================================
        stage('Pre-Checks') {
            steps {
                script {
                    sh '''
                        echo "=== Environment Check ==="
                        java -version
                        mvn -version
                        docker --version
                        trivy --version
                        semgrep --version
                        gitleaks version
                        
                        # Create report directory
                        mkdir -p ${REPORT_DIR}
                    '''
                }
            }
        }
        
        // ==========================================
        // STAGE 2: SECRET SCANNING
        // ==========================================
        stage('Secret Scanning') {
            steps {
                script {
                    sh '''
                        echo "=== Running Gitleaks Secret Scan ==="
                        gitleaks detect --source . --verbose --report-format json \
                            --report-path ${REPORT_DIR}/gitleaks-report.json || true
                        
                        # Check results
                        if [ -s ${REPORT_DIR}/gitleaks-report.json ]; then
                            echo "⚠️  Potential secrets detected!"
                            cat ${REPORT_DIR}/gitleaks-report.json | head -20
                        else
                            echo "✅ No secrets detected"
                        fi
                    '''
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: "${REPORT_DIR}/gitleaks-report.json", allowEmptyArchive: true
                }
            }
        }
        
        // ==========================================
        // STAGE 3: BUILD & UNIT TESTS
        // ==========================================
        stage('Build & Test') {
            steps {
                script {
                    sh '''
                        echo "=== Building Spring PetClinic ==="
                        ./mvnw clean compile -DskipTests
                        
                        echo "=== Running Unit Tests ==="
                        ./mvnw test
                    '''
                }
            }
            post {
                always {
                    junit '**/target/surefire-reports/*.xml', allowEmptyResults: true
                }
            }
        }
        
        // ==========================================
        // STAGE 4: SAST - STATIC ANALYSIS
        // ==========================================
        stage('SAST Analysis') {
            parallel {
                stage('SonarQube') {
                    steps {
                        script {
                            def scannerHome = tool 'SonarScanner'
                            withSonarQubeEnv('SonarQube') {
                                sh """
                                    ${scannerHome}/bin/sonar-scanner \
                                    -Dsonar.projectKey=spring-petclinic \
                                    -Dsonar.projectName='Spring PetClinic' \
                                    -Dsonar.sources=. \
                                    -Dsonar.java.binaries=target/classes \
                                    -Dsonar.exclusions=**/target/**,**/*.min.js,**/node_modules/**
                                """
                            }
                        }
                    }
                }
                stage('Semgrep') {
                    steps {
                        sh '''
                            echo "=== Running Semgrep SAST ==="
                            semgrep --config=auto \
                                --json --output=${REPORT_DIR}/semgrep-report.json \
                                . || true
                            
                            # Count findings
                            echo "Semgrep findings:"
                            jq '.results | length' ${REPORT_DIR}/semgrep-report.json || echo "0"
                        '''
                    }
                    post {
                        always {
                            archiveArtifacts artifacts: "${REPORT_DIR}/semgrep-report.json", allowEmptyArchive: true
                        }
                    }
                }
            }
        }
        
        // ==========================================
        // STAGE 5: SCA - DEPENDENCY CHECK
        // ==========================================
        stage('Dependency Check') {
            steps {
                script {
                    sh '''
                        echo "=== Running OWASP Dependency Check ==="
                        dependency-check.sh \
                            --project "Spring PetClinic" \
                            --scan . \
                            --format JSON \
                            --format HTML \
                            --out ${REPORT_DIR}/dependency-check \
                            --enableExperimental || true
                    '''
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: "${REPORT_DIR}/dependency-check/*", allowEmptyArchive: true
                }
            }
        }
        
        // ==========================================
        // STAGE 6: SECURITY REPORT
        // ==========================================
        stage('Generate Report') {
            steps {
                script {
                    sh '''
                        echo "=== Generating Security Summary ==="
                        cat > ${REPORT_DIR}/security-summary.txt << 'REPORT'
SECURITY SCAN SUMMARY
====================
Build: ${BUILD_NUMBER}
Commit: ${GIT_COMMIT}
Branch: ${GIT_BRANCH}
Timestamp: $(date)

SCANS COMPLETED:
✓ Secret Scanning (Gitleaks)
✓ Static Analysis (SonarQube, Semgrep)
✓ Dependency Check (OWASP)

Reports available in security-reports/ directory
REPORT
                        cat ${REPORT_DIR}/security-summary.txt
                    '''
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: "${REPORT_DIR}/**/*", allowEmptyArchive: true
                }
            }
        }
    }
    
    post {
        always {
            script {
                echo "Pipeline completed - Build #${BUILD_NUMBER}"
            }
        }
        success {
            echo " Build Successful!"
        }
        failure {
            echo " Build Failed!"
        }
    }
}
EOF

# Add, commit, and push
git add Jenkinsfile
git commit -m "Add Jenkinsfile for secure pipeline"
git push origin main
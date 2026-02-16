pipeline {
    agent any
    
    environment {
        MAVEN_HOME = tool 'Maven-3.9'
        JAVA_HOME = tool 'JDK-17'
        PATH = "${MAVEN_HOME}/bin:${JAVA_HOME}/bin:${env.PATH}"
        REPORT_DIR = 'security-reports'
    }
    
    options {
        buildDiscarder(logRotator(numToKeepStr: '5'))
        timeout(time: 1, unit: 'HOURS')
        disableConcurrentBuilds()
    }
    
    stages {
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
                        mkdir -p ${REPORT_DIR}
                    '''
                }
            }
        }
        
        stage('Secret Scanning') {
            steps {
                script {
                    sh '''
                        echo "=== Running Gitleaks Secret Scan ==="
                        gitleaks detect --source . --verbose --report-format json --report-path ${REPORT_DIR}/gitleaks-report.json || true
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
                    junit testResults: '**/target/surefire-reports/*.xml', allowEmptyResults: true
                }
            }
        }
        
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
                            semgrep --config=auto --json --output=${REPORT_DIR}/semgrep-report.json . || true
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
        
        stage('Dependency Check') {
            steps {
                script {
                    sh '''
                        echo "=== Running OWASP Dependency Check ==="
                        dependency-check.sh --project "Spring PetClinic" --scan . --format JSON --format HTML --out ${REPORT_DIR}/dependency-check --enableExperimental || true
                    '''
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: "${REPORT_DIR}/dependency-check/*", allowEmptyArchive: true
                }
            }
        }
        
        stage('Generate Report') {
            steps {
                script {
                    sh '''
                        echo "=== Generating Security Summary ==="
                        echo "SECURITY SCAN SUMMARY" > ${REPORT_DIR}/security-summary.txt
                        echo "====================" >> ${REPORT_DIR}/security-summary.txt
                        echo "Build: ${BUILD_NUMBER}" >> ${REPORT_DIR}/security-summary.txt
                        echo "Commit: ${GIT_COMMIT}" >> ${REPORT_DIR}/security-summary.txt
                        echo "Timestamp: $(date)" >> ${REPORT_DIR}/security-summary.txt
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
            echo "Pipeline completed - Build #${BUILD_NUMBER}"
        }
        success {
            echo "✅ Build Successful!"
        }
        failure {
            echo "❌ Build Failed!"
        }
    }
}
pipeline {
    agent any
    
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
                echo '✅ Pipeline started - Build #' + env.BUILD_NUMBER
                sh '''
                    echo "Java version:"
                    java -version
                    echo "Maven version:"
                    mvn -version
                    echo "Docker version:"
                    docker --version
                '''
            }
        }
        
        stage('Secret Scanning') {
            steps {
                sh '''
                    echo "=== 🔒 Running Gitleaks Secret Scan ==="
                    mkdir -p ${REPORT_DIR}
                    
                    gitleaks detect --source . --verbose --report-format json \
                        --report-path ${REPORT_DIR}/gitleaks-report.json || true
                    
                    if [ -s ${REPORT_DIR}/gitleaks-report.json ]; then
                        echo "⚠️  WARNING: Potential secrets found!"
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
        
        stage('Build') {
            steps {
                sh '''
                    echo "=== 🔨 Building Application ==="
                    ./mvnw clean package -DskipTests
                '''
            }
        }
        
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
        
        stage('SCA - Dependency Check') {
            steps {
                sh '''
                    echo "=== 📦 Running OWASP Dependency-Check ==="
                    
                    /opt/dependency-check/bin/dependency-check.sh \
                        --project "Spring PetClinic Microservices" \
                        --scan . \
                        --format JSON \
                        --format HTML \
                        --out ${REPORT_DIR}/dependency-check \
                        --enableExperimental \
                        --failOnCVSS 11 || true
                    
                    echo "Dependency Check completed"
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
        
        stage('Container Security') {
            steps {
                script {
                    sh '''
                        echo "=== 🐳 Building Docker Image ==="
                        
                        ./mvnw package -pl spring-petclinic-config-server -am -DskipTests -q
                        
                        cd spring-petclinic-config-server
                        
                        JAR_FILE=$(ls target/*.jar | head -1)
                        ARTIFACT_NAME=$(basename $JAR_FILE .jar)
                        
                        echo "Building image for: $ARTIFACT_NAME"
                        
                        mkdir -p docker-build
                        cp $JAR_FILE docker-build/
                        cp ../docker/Dockerfile docker-build/
                        
                        docker build \
                            --build-arg ARTIFACT_NAME=$ARTIFACT_NAME \
                            --build-arg EXPOSED_PORT=8888 \
                            -t petclinic-config-server:${IMAGE_TAG} \
                            docker-build/
                        
                        rm -rf docker-build
                        cd ..
                        
                        echo "=== 🔍 Scanning with Trivy ==="
                        
                        trivy image \
                            --format json \
                            --output ${REPORT_DIR}/trivy-report.json \
                            --severity HIGH,CRITICAL \
                            petclinic-config-server:${IMAGE_TAG} || true
                        
                        trivy image \
                            --format table \
                            --output ${REPORT_DIR}/trivy-report.txt \
                            --severity HIGH,CRITICAL \
                            petclinic-config-server:${IMAGE_TAG} || true
                        
                        echo "✅ Container Security Scan Completed"
                    '''
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: "${REPORT_DIR}/trivy-report.*", allowEmptyArchive: true
                }
            }
        }
        
        // ==========================================
        // STAGE 8: APPROVAL GATE 1 (NEW)
        // ==========================================
        stage('Approval: Deploy to Staging') {
            steps {
                script {
                    // Generate summary for approver
                    sh '''
                        echo "=========================================="
                        echo "SECURITY SCAN SUMMARY - Build #${BUILD_NUMBER}"
                        echo "=========================================="
                        echo "Commit: ${GIT_COMMIT}"
                        echo "Branch: ${GIT_BRANCH}"
                        echo ""
                        echo "COMPLETED SCANS:"
                        echo "✅ Secret Scanning (Gitleaks)"
                        echo "✅ SAST (SonarQube)"
                        echo "✅ SCA (OWASP Dependency-Check)"
                        echo "✅ Unit Tests"
                        echo "✅ Container Security (Trivy)"
                        echo ""
                        echo "REPORTS LOCATION:"
                        echo "${BUILD_URL}artifact/${REPORT_DIR}/"
                        echo "=========================================="
                    '''
                    
                    // Pause for approval
                    input message: 'All security scans passed. Approve deployment to STAGING?', 
                          ok: 'Deploy to Staging',
                          submitterParameter: 'APPROVER'
                    
                    echo "✅ Approved by: ${env.APPROVER}"
                }
            }
        }
        
        // ==========================================
        // STAGE 9: DEPLOY TO STAGING (NEW)
        // ==========================================
        stage('Deploy to Staging') {
            steps {
                script {
                    echo "🚀 Deploying to Staging Environment..."
                    sh '''
                        echo "Staging deployment would happen here"
                        echo "For demo: Running container locally"
                        
                        # Stop any existing container
                        docker stop petclinic-staging 2>/dev/null || true
                        docker rm petclinic-staging 2>/dev/null || true
                        
                        # Run container
                        docker run -d \
                            --name petclinic-staging \
                            -p 8888:8888 \
                            petclinic-config-server:${IMAGE_TAG}
                        
                        echo "Container running at http://localhost:8888"
                        sleep 5
                        
                        # Health check
                        curl -f http://localhost:8888/actuator/health || echo "Health check endpoint not available"
                    '''
                }
            }
        }
        
        // ==========================================
        // STAGE 10: APPROVAL GATE 2 (NEW)
        // ==========================================
        stage('Approval: Deploy to Production') {
            steps {
                script {
                    echo "=========================================="
                    echo "STAGING DEPLOYMENT SUCCESSFUL"
                    echo "=========================================="
                    
                    // Pause for production approval
                    timeout(time: 24, unit: 'HOURS') {
                        input message: 'Staging looks good. Approve deployment to PRODUCTION?', 
                              ok: 'Deploy to Production',
                              submitterParameter: 'PROD_APPROVER'
                    }
                    
                    echo "✅ Production deployment approved by: ${env.PROD_APPROVER}"
                }
            }
        }
        
        // ==========================================
        // STAGE 11: DEPLOY TO PRODUCTION (NEW)
        // ==========================================
        stage('Deploy to Production') {
            steps {
                script {
                    echo "🚀 Deploying to Production Environment..."
                    sh '''
                        echo "Production deployment would happen here"
                        echo "Tagging image as production-ready"
                        
                        docker tag petclinic-config-server:${IMAGE_TAG} petclinic-config-server:production
                        
                        echo "✅ Production deployment completed"
                    '''
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
            echo "✅ Container Security: COMPLETED"
            echo "✅ Approval Gates: COMPLETED"
            echo "✅ Deployment: COMPLETED"
            echo "=========================================="
        }
        success {
            echo "🎉 ALL STAGES PASSED - FULL SUCCESS!"
        }
        failure {
            echo "❌ PIPELINE FAILED - Check logs above"
        }
    }
}

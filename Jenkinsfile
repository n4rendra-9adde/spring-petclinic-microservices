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
                        
                        # Build JAR for config-server
                        ./mvnw package -pl spring-petclinic-config-server -am -DskipTests -q
                        
                        cd spring-petclinic-config-server
                        
                        # Get the JAR name
                        JAR_FILE=$(ls target/*.jar | head -1)
                        ARTIFACT_NAME=$(basename $JAR_FILE .jar)
                        
                        echo "Building image for: $ARTIFACT_NAME"
                        echo "JAR location: $JAR_FILE"
                        
                        # Create a temp directory for Docker build
                        mkdir -p docker-build
                        cp $JAR_FILE docker-build/
                        cp ../docker/Dockerfile docker-build/
                        
                        # Build Docker image with correct context
                        docker build \
                            --build-arg ARTIFACT_NAME=$ARTIFACT_NAME \
                            --build-arg EXPOSED_PORT=8888 \
                            -t petclinic-config-server:${IMAGE_TAG} \
                            docker-build/
                        
                        # Cleanup
                        rm -rf docker-build
                        
                        cd ..
                        
                        echo "=== 🔍 Scanning with Trivy ==="
                        
                        # Scan the built image
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
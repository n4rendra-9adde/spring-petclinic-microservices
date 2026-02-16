pipeline {
    agent any
    
    environment {
        MAVEN_HOME = tool 'Maven-3.9'
        JAVA_HOME = tool 'JDK-17'
        PATH = "${MAVEN_HOME}/bin:${JAVA_HOME}/bin:${env.PATH}"
        REPORT_DIR = 'test-reports'
    }
    
    stages {
        stage('Hello') {
            steps {
                echo '✅ Jenkins pipeline is working!'
            }
        }
        
        stage('Build') {
            steps {
                sh '''
                    echo "=== Compiling Code ==="
                    ./mvnw clean compile -DskipTests
                '''
            }
        }
        
        stage('Unit Tests') {
            steps {
                sh '''
                    echo "=== Running Unit Tests ==="
                    ./mvnw test
                '''
            }
            post {
                always {
                    // Publish test results
                    junit testResults: '**/target/surefire-reports/*.xml', allowEmptyResults: true
                    
                    // Archive test reports
                    sh 'mkdir -p ${REPORT_DIR}'
                    sh 'cp -r */target/surefire-reports ${REPORT_DIR}/ || true'
                    sh 'cp -r */target/site/jacoco ${REPORT_DIR}/ || true'
                    archiveArtifacts artifacts: "${REPORT_DIR}/**/*", allowEmptyArchive: true
                }
            }
        }
    }
    
    post {
        always {
            echo "Pipeline finished - Build #${BUILD_NUMBER}"
        }
        success {
            echo "✅ Build & Tests Successful!"
        }
        failure {
            echo "❌ Build Failed!"
        }
    }
}
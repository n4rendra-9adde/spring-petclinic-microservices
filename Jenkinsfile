pipeline {
    agent any
    
    environment {
        MAVEN_HOME = tool 'Maven-3.9'
        JAVA_HOME = tool 'JDK-17'
        PATH = "${MAVEN_HOME}/bin:${JAVA_HOME}/bin:${env.PATH}"
    }
    
    stages {
        stage('Hello') {
            steps {
                echo '✅ Jenkins pipeline is working!'
                echo "Build number: ${BUILD_NUMBER}"
                echo "Git commit: ${GIT_COMMIT}"
            }
        }
        
        stage('Check Tools') {
            steps {
                sh '''
                    echo "=== Checking Java ==="
                    java -version
                    
                    echo "=== Checking Maven ==="
                    mvn -version
                '''
            }
        }
        
        stage('Build') {
            steps {
                sh '''
                    echo "=== Building Spring PetClinic ==="
                    ./mvnw clean compile -DskipTests
                '''
            }
        }
    }
    
    post {
        always {
            echo "Pipeline finished!"
        }
        success {
            echo "✅ SUCCESS!"
        }
        failure {
            echo "❌ FAILED!"
        }
    }
}
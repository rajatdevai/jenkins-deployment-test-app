pipeline {
    agent any
    
    tools {
        maven 'Maven 3.9.11'    // Uses the Maven tool you configured
        jdk 'JDK 17'
    }
    
    parameters {
        choice(
            name: 'ENVIRONMENT',
            choices: ['sandbox', 'test', 'prod'],
            description: 'Select deployment environment'
        )
        booleanParam(
            name: 'SKIP_TESTS',
            defaultValue: false,
            description: 'Skip tests to speed up build'
        )
    }
    
    environment {
        CONNECTED_APP_ID = credentials('connected-app-id')
        CONNECTED_APP_SECRET = credentials('connected-app-secret')
    }
    
    stages {
        stage('Verify Maven & Java') {
            steps {
                bat '''
                    echo.
                    echo ===== MAVEN CHECK =====
                    mvn --version
                    echo.
                    echo ===== JAVA CHECK =====
                    java -version
                    echo.
                '''
            }
        }
        
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        
        stage('Build') {
            steps {
                script {
                    def skipTests = params.SKIP_TESTS ? '-DskipTests' : ''
                    bat """
                        echo Building with profile: profile-%ENVIRONMENT%
                        mvn clean package -Pprofile-%ENVIRONMENT% ${skipTests}
                    """
                }
            }
        }
        
        stage('Publish to Exchange') {
            steps {
                bat '''
                    echo Publishing to Anypoint Exchange...
                    mvn deploy -Pprofile-%ENVIRONMENT% -s settings.xml -DskipTests
                '''
            }
        }
        
        stage('Deploy to CloudHub') {
            steps {
                bat '''
                    echo Deploying to CloudHub 2.0 with profile: profile-%ENVIRONMENT%
                    echo Environment: %ENVIRONMENT%
                    mvn deploy -DmuleDeploy -Pprofile-%ENVIRONMENT% -DskipTests
                '''
            }
        }
    }
    
    post {
        success {
            echo "✅ Build and Deployment Successful!"
        }
        failure {
            echo "❌ Build Failed - Check logs above"
        }
        always {
            cleanWs()
        }
    }
}
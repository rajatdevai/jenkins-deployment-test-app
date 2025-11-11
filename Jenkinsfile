// ====================================================================
// JENKINS DECLARATIVE PIPELINE FOR MULE CLOUDHUB DEPLOYMENT
// ====================================================================

pipeline {
    
    agent {
        // Run on any available agent with 'mule' label
        node {
            label 'mule'
        }
    }
    
    // ===== ENVIRONMENT VARIABLES =====
    environment {
        // Maven Configuration
        MAVEN_HOME = 'C:/Program Files/apache-maven-3.9.11'
        JAVA_HOME = 'C:/Program Files/Java/jdk-17'
        PATH = "${MAVEN_HOME}/bin:${JAVA_HOME}/bin:${PATH}"
        
        // MuleSoft Anypoint Credentials (from Jenkins Secrets)
        ANYPOINT_USERNAME = credentials('anypoint-username')
        ANYPOINT_PASSWORD = credentials('anypoint-password')
        
        // Build Configuration
        BUILD_NAME = "${JOB_NAME}-${BUILD_NUMBER}"
        TIMESTAMP = sh(script: "date +%Y%m%d_%H%M%S", returnStdout: true).trim()
        
        // Application Artifact
        APP_JAR = "target/${PROJECT_ARTIFACT}-${PROJECT_VERSION}.jar"
    }
    
    // ===== PARAMETERS (User Input) =====
    parameters {
        choice(
            name: 'DEPLOYMENT_ENV',
            choices: ['Sandbox', 'Test', 'prod'],
            description: 'Select deployment environment'
        )
        
        string(
            name: 'APP_NAME',
            defaultValue: 'munit-test-cases',
            description: 'Application name in CloudHub'
        )
        
        string(
            name: 'REPLICAS',
            defaultValue: '1',
            description: 'Number of replicas (1, 2, etc.)'
        )
        
        string(
            name: 'VCORES',
            defaultValue: '0.1',
            description: 'vCores per replica (0.1, 0.2, 1, 2)'
        )
        
        booleanParam(
            name: 'RUN_TESTS',
            defaultValue: true,
            description: 'Run MUnit tests before deployment?'
        )
        
        booleanParam(
            name: 'SKIP_DEPLOY',
            defaultValue: false,
            description: 'Skip deployment (build & test only)?'
        )
    }
    
    // ===== BUILD TRIGGERS =====
    triggers {
        // Trigger on GitHub webhook push
        githubPush()
        
        // Poll SCM every 15 minutes (fallback if webhook fails)
        pollSCM('H/15 * * * *')
    }
    
    // ===== BUILD OPTIONS =====
    options {
        // Keep last 10 builds
        buildDiscarder(logRotator(numToKeepStr: '10', artifactNumToKeepStr: '5'))
        
        // Build timeout: 1 hour
        timeout(time: 1, unit: 'HOURS')
        
        // Print timestamps in logs
        timestamps()
        
        // Colorize console output
        ansiColor('xterm')
        
        // Prevent concurrent builds
        disableConcurrentBuilds()
    }
    
    // ===== PIPELINE STAGES =====
    stages {
        
        // ===== STAGE 1: INITIALIZATION =====
        stage('Initialize') {
            steps {
                script {
                    echo "╔════════════════════════════════════════╗"
                    echo "║        JENKINS PIPELINE STARTED         ║"
                    echo "╠════════════════════════════════════════╣"
                    echo "  Build Name: ${BUILD_NAME}"
                    echo "  Timestamp: ${TIMESTAMP}"
                    echo "  Environment: ${params.DEPLOYMENT_ENV}"
                    echo "  App Name: ${params.APP_NAME}"
                    echo "  Replicas: ${params.REPLICAS}"
                    echo "  vCores: ${params.VCORES}"
                    echo "  Run Tests: ${params.RUN_TESTS}"
                    echo "  Skip Deploy: ${params.SKIP_DEPLOY}"
                    echo "╚════════════════════════════════════════╝"
                    
                    // Validate parameters
                    if (!params.DEPLOYMENT_ENV) {
                        error("ERROR: Deployment environment not selected!")
                    }
                    
                    echo "✅ All parameters validated successfully"
                }
            }
        }
        
        // ===== STAGE 2: CHECKOUT =====
        stage('Checkout') {
            steps {
                script {
                    echo "📂 Checking out source code from repository..."
                    
                    try {
                        // Clone from GitHub
                        checkout([
                            $class: 'GitSCM',
                            branches: [[name: '*/prod']],
                            userRemoteConfigs: [
                                [url: 'https://github.com/rajatdevai/jenkins-deployment-test-app.git']
                            ]
                        ])
                        
                        echo "✅ Checkout successful"
                        
                        // Display checked out branch and commit
                        sh '''
                            echo "📋 Git Status:"
                            git log -1 --oneline
                            echo ""
                            echo "🔗 Branch: $(git rev-parse --abbrev-ref HEAD)"
                            echo "📍 Commit Hash: $(git rev-parse HEAD)"
                        '''
                    } catch (Exception e) {
                        echo "❌ Checkout failed: ${e.message}"
                        error("Failed to checkout source code")
                    }
                }
            }
        }
        
        // ===== STAGE 3: BUILD =====
        stage('Build') {
            steps {
                script {
                    echo "🔨 Building Mule application..."
                    
                    try {
                        // Clean and compile
                        sh '''
                            echo "Cleaning previous build artifacts..."
                            ${MAVEN_HOME}/bin/mvn clean
                            
                            echo ""
                            echo "Compiling Mule application..."
                            ${MAVEN_HOME}/bin/mvn compile \
                                -Dorg.slf4j.simpleLogger.defaultLogLevel=info \
                                -B
                            
                            echo "✅ Build successful"
                        '''
                    } catch (Exception e) {
                        echo "❌ Build failed: ${e.message}"
                        error("Maven build failed")
                    }
                }
            }
        }
        
        // ===== STAGE 4: UNIT TESTS =====
        stage('Run MUnit Tests') {
            when {
                expression { params.RUN_TESTS == true }
            }
            steps {
                script {
                    echo "🧪 Running MUnit tests..."
                    
                    try {
                        sh '''
                            ${MAVEN_HOME}/bin/mvn test \
                                -Dorg.slf4j.simpleLogger.defaultLogLevel=info \
                                -B
                            
                            echo "✅ All tests passed"
                        '''
                    } catch (Exception e) {
                        echo "❌ Tests failed: ${e.message}"
                        
                        // Always publish test results even if tests fail
                        publishHTML([
                            reportDir: 'target/munit-reports',
                            reportFiles: 'index.html',
                            reportName: 'MUnit Test Report'
                        ])
                        
                        error("MUnit tests failed - see report above")
                    }
                }
            }
            post {
                always {
                    // Publish MUnit test results
                    junit 'target/munit-reports/**/*.xml'
                    
                    // Publish HTML test report
                    publishHTML([
                        reportDir: 'target/munit-reports',
                        reportFiles: 'index.html',
                        reportName: 'MUnit Test Report',
                        keepAll: true
                    ])
                    
                    echo "📊 Test reports published"
                }
            }
        }
        
        // ===== STAGE 5: PACKAGE =====
        stage('Package') {
            steps {
                script {
                    echo "📦 Packaging application..."
                    
                    try {
                        sh '''
                            ${MAVEN_HOME}/bin/mvn package \
                                -DskipTests \
                                -Dorg.slf4j.simpleLogger.defaultLogLevel=info \
                                -B
                            
                            echo ""
                            echo "Artifact created:"
                            ls -lh target/*.jar
                            
                            echo "✅ Package successful"
                        '''
                    } catch (Exception e) {
                        echo "❌ Package failed: ${e.message}"
                        error("Maven package failed")
                    }
                }
            }
        }
        
        // ===== STAGE 6: DEPLOY TO CLOUDHUB =====
        stage('Deploy to CloudHub') {
            when {
                expression { params.SKIP_DEPLOY == false }
            }
            steps {
                script {
                    echo "🚀 Deploying to CloudHub (${params.DEPLOYMENT_ENV})..."
                    
                    try {
                        // Determine Maven profile based on environment
                        def mavenProfile = mapEnvironmentToProfile(params.DEPLOYMENT_ENV)
                        
                        echo "Using Maven profile: ${mavenProfile}"
                        
                        sh '''
                            # Export credentials for Maven
                            export MAVEN_OPTS="-Xmx1024m"
                            
                            ${MAVEN_HOME}/bin/mvn deploy \
                                -P ${MAVEN_PROFILE} \
                                -Danypoint.username="${ANYPOINT_USERNAME}" \
                                -Danypoint.password="${ANYPOINT_PASSWORD}" \
                                -Ddeployment.app.name="${APP_NAME}" \
                                -Denv.cloudHub.replicas="${REPLICAS}" \
                                -Denv.cloudHub.vcores="${VCORES}" \
                                -DskipTests \
                                -Dorg.slf4j.simpleLogger.defaultLogLevel=info \
                                -B
                            
                            echo "✅ Deployment successful"
                        '''
                    } catch (Exception e) {
                        echo "❌ Deployment failed: ${e.message}"
                        error("Deployment to CloudHub failed")
                    }
                }
            }
        }
        
        // ===== STAGE 7: VERIFY DEPLOYMENT =====
        stage('Verify Deployment') {
            when {
                expression { params.SKIP_DEPLOY == false }
            }
            steps {
                script {
                    echo "✔️ Verifying deployment..."
                    
                    try {
                        sh '''
                            echo "Waiting for application to start (30 seconds)..."
                            sleep 30
                            
                            echo "Checking application health..."
                            
                            # This would check CloudHub API for app status
                            # Replace with your actual health check logic
                            
                            echo "Deployment verification complete"
                        '''
                    } catch (Exception e) {
                        echo "⚠️ Verification check failed: ${e.message}"
                        // Don't fail the build on verification failure
                    }
                }
            }
        }
    }
    
    // ===== POST BUILD ACTIONS =====
    post {
        always {
            script {
                echo "🧹 Cleaning up..."
                
                // Archive build artifacts
                archiveArtifacts artifacts: 'target/*.jar', 
                                 allowEmptyArchive: true,
                                 onlyIfSuccessful: false
                
                // Clean workspace
                cleanWs()
                
                echo "✅ Cleanup complete"
            }
        }
        
        success {
            script {
                echo "✅ PIPELINE SUCCESSFUL!"
                echo ""
                echo "╔════════════════════════════════════════╗"
                echo "║        DEPLOYMENT SUCCESSFUL           ║"
                echo "╠════════════════════════════════════════╣"
                echo "  Build: ${BUILD_NAME}"
                echo "  Environment: ${params.DEPLOYMENT_ENV}"
                echo "  App Name: ${params.APP_NAME}"
                echo "  Duration: ${currentBuild.durationString}"
                echo "╚════════════════════════════════════════╝"
                
                // Send success notification
                sendNotification('SUCCESS', 'Deployment completed successfully!')
            }
        }
        
        failure {
            script {
                echo "❌ PIPELINE FAILED!"
                echo ""
                echo "╔════════════════════════════════════════╗"
                echo "║         DEPLOYMENT FAILED              ║"
                echo "╠════════════════════════════════════════╣"
                echo "  Build: ${BUILD_NAME}"
                echo "  Environment: ${params.DEPLOYMENT_ENV}"
                echo "  Check logs above for details"
                echo "╚════════════════════════════════════════╝"
                
                // Send failure notification
                sendNotification('FAILURE', 'Deployment failed! Check logs.')
            }
        }
        
        unstable {
            script {
                echo "⚠️ PIPELINE UNSTABLE (Tests Failed)"
                sendNotification('UNSTABLE', 'Pipeline unstable! Some tests failed.')
            }
        }
    }
}

// ====================================================================
// HELPER FUNCTIONS
// ====================================================================

/**
 * Maps environment name to Maven profile
 */
def mapEnvironmentToProfile(String environment) {
    switch(environment) {
        case 'Sandbox':
            return 'profile-sandbox'
        case 'Test':
            return 'profile-test'
        case 'Production':
            return 'profile-prod'
        default:
            error("Unknown environment: ${environment}")
    }
}

/**
 * Sends notification (Slack, Email, etc.)
 */
def sendNotification(String status, String message) {
    // Email notification
    emailext(
        subject: "[Jenkins] Mule App Deployment - ${status}",
        body: """
            Build: ${BUILD_NAME}
            Status: ${status}
            Message: ${message}
            
            Build Log: ${BUILD_URL}console
        """,
        to: 'your-email@company.com',
        mimeType: 'text/plain'
    )
    
    // Slack notification (if configured)
    try {
        def slackColor = status == 'SUCCESS' ? 'good' : 'danger'
        slackSend(
            color: slackColor,
            message: "${status}: Mule App Deployment\n${message}\nBuild: ${BUILD_URL}"
        )
    } catch (Exception e) {
        echo "Slack notification failed: ${e.message}"
    }
}
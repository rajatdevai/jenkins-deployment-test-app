// ====================================================================
// JENKINS DECLARATIVE PIPELINE FOR MULE CLOUDHUB 2.0 DEPLOYMENT
// WITH ANYPOINT EXCHANGE PUBLICATION - MINIMAL VERSION
// ====================================================================

pipeline {
    
    agent any
    
    // ===== ENVIRONMENT VARIABLES =====
    environment {
        // Maven Configuration (Windows paths)
        MAVEN_HOME = 'C:/Program Files/apache-maven-3.9.11'
        JAVA_HOME = 'C:/Program Files/Java/jdk-17'
        PATH = "${MAVEN_HOME}/bin;${JAVA_HOME}/bin;${env.PATH}"
        
        // MuleSoft Anypoint Connected App Credentials (from Jenkins Secrets)
        CONNECTED_APP_ID = credentials('connected-app-id')
        CONNECTED_APP_SECRET = credentials('connected-app-secret')
        
        // Organization ID (from Jenkins Configuration)
        ANYPOINT_ORG_ID = credentials('anypoint-org-id')
        
        // Build Configuration
        BUILD_NAME = "${JOB_NAME}-${BUILD_NUMBER}"
    }
    
    // ===== PARAMETERS (User Input) =====
    parameters {
        choice(
            name: 'DEPLOYMENT_ENV',
            choices: ['Sandbox', 'Test', 'Production'],
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
            name: 'PUBLISH_TO_EXCHANGE',
            defaultValue: true,
            description: 'Publish to Anypoint Exchange?'
        )
        
        booleanParam(
            name: 'SKIP_DEPLOY',
            defaultValue: false,
            description: 'Skip deployment (build & test only)?'
        )
        
        string(
            name: 'EXCHANGE_ASSET_VERSION',
            defaultValue: '1.0.0',
            description: 'Asset version for Exchange (e.g., 1.0.0, 1.0.1)'
        )
    }
    
    // ===== BUILD OPTIONS =====
    options {
        // Keep last 10 builds
        buildDiscarder(logRotator(numToKeepStr: '10', artifactNumToKeepStr: '5'))
        
        // Build timeout: 1 hour
        timeout(time: 1, unit: 'HOURS')
        
        // Print timestamps in logs
        timestamps()
        
        // Prevent concurrent builds
        disableConcurrentBuilds()
    }
    
    // ===== PIPELINE STAGES =====
    stages {
        
        // ===== STAGE 1: INITIALIZATION =====
        stage('Initialize') {
            steps {
                script {
                    echo "========================================"
                    echo "    JENKINS PIPELINE STARTED"
                    echo "========================================"
                    echo "Build Name: ${BUILD_NAME}"
                    echo "Environment: ${params.DEPLOYMENT_ENV}"
                    echo "App Name: ${params.APP_NAME}"
                    echo "Replicas: ${params.REPLICAS}"
                    echo "vCores: ${params.VCORES}"
                    echo "Run Tests: ${params.RUN_TESTS}"
                    echo "Publish to Exchange: ${params.PUBLISH_TO_EXCHANGE}"
                    echo "Exchange Version: ${params.EXCHANGE_ASSET_VERSION}"
                    echo "Skip Deploy: ${params.SKIP_DEPLOY}"
                    echo "========================================"
                    
                    // Validate parameters
                    if (!params.DEPLOYMENT_ENV) {
                        error("ERROR: Deployment environment not selected!")
                    }
                    
                    // Validate version format
                    if (!params.EXCHANGE_ASSET_VERSION.matches(/^\d+\.\d+\.\d+$/)) {
                        error("ERROR: Invalid version format! Use semantic versioning (e.g., 1.0.0)")
                    }
                    
                    echo "All parameters validated successfully"
                }
            }
        }
        
        // ===== STAGE 2: CHECKOUT =====
        stage('Checkout') {
            steps {
                script {
                    echo "Checking out source code from repository..."
                    
                    try {
                        // Clone from GitHub
                        checkout([
                            $class: 'GitSCM',
                            branches: [[name: '*/main']],
                            userRemoteConfigs: [
                                [url: 'https://github.com/rajatdevai/jenkins-deployment-test-app.git']
                            ]
                        ])
                        
                        echo "Checkout successful"
                        
                        // Display checked out branch and commit (Windows compatible)
                        bat '''
                            echo Git Status:
                            git log -1 --oneline
                            echo.
                            git rev-parse --abbrev-ref HEAD
                            git rev-parse HEAD
                        '''
                    } catch (Exception e) {
                        echo "Checkout failed: ${e.message}"
                        error("Failed to checkout source code")
                    }
                }
            }
        }
        
        // ===== STAGE 3: BUILD =====
        stage('Build') {
            steps {
                script {
                    echo "Building Mule application..."
                    
                    try {
                        // Clean and compile (Windows)
                        bat '''
                            echo Cleaning previous build artifacts...
                            "%MAVEN_HOME%/bin/mvn" clean
                            
                            echo.
                            echo Compiling Mule application...
                            "%MAVEN_HOME%/bin/mvn" compile -Dorg.slf4j.simpleLogger.defaultLogLevel=info -B
                            
                            echo Build successful
                        '''
                    } catch (Exception e) {
                        echo "Build failed: ${e.message}"
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
                    echo "Running MUnit tests..."
                    
                    try {
                        bat '''
                            "%MAVEN_HOME%/bin/mvn" test -Dorg.slf4j.simpleLogger.defaultLogLevel=info -B
                            
                            echo All tests passed
                        '''
                    } catch (Exception e) {
                        echo "Tests failed: ${e.message}"
                        error("MUnit tests failed")
                    }
                }
            }
            post {
                always {
                    // Publish MUnit test results
                    junit allowEmptyResults: true, testResults: 'target/munit-reports/**/*.xml'
                    
                    // Publish HTML test report
                    publishHTML([
                        reportDir: 'target/munit-reports',
                        reportFiles: 'index.html',
                        reportName: 'MUnit Test Report',
                        keepAll: true,
                        allowMissing: true
                    ])
                    
                    echo "Test reports published"
                }
            }
        }
        
        // ===== STAGE 5: PACKAGE =====
        stage('Package') {
            steps {
                script {
                    echo "Packaging application..."
                    
                    try {
                        bat '''
                            "%MAVEN_HOME%/bin/mvn" package -DskipTests -Dorg.slf4j.simpleLogger.defaultLogLevel=info -B
                            
                            echo.
                            echo Artifact created:
                            dir target\\*.jar
                            
                            echo Package successful
                        '''
                    } catch (Exception e) {
                        echo "Package failed: ${e.message}"
                        error("Maven package failed")
                    }
                }
            }
        }
        
        // ===== STAGE 6: PUBLISH TO ANYPOINT EXCHANGE =====
        stage('Publish to Exchange') {
            when {
                expression { params.PUBLISH_TO_EXCHANGE == true }
            }
            steps {
                script {
                    echo "Publishing to Anypoint Exchange..."
                    echo "Asset Version: ${params.EXCHANGE_ASSET_VERSION}"
                    
                    try {
                        bat """
                            @echo off
                            
                            REM Set version in POM
                            "%MAVEN_HOME%/bin/mvn" versions:set -DnewVersion=${params.EXCHANGE_ASSET_VERSION}
                            
                            echo.
                            echo Publishing to Exchange...
                            
                            REM Deploy to Exchange
                            "%MAVEN_HOME%/bin/mvn" deploy ^
                                -DskipTests ^
                                -DaltDeploymentRepository=anypoint-exchange-v3::default::https://maven.anypoint.mulesoft.com/api/v3/organizations/${ANYPOINT_ORG_ID}/maven ^
                                -Dorg.slf4j.simpleLogger.defaultLogLevel=info ^
                                -B
                            
                            echo Successfully published to Anypoint Exchange
                            echo Asset: ${params.APP_NAME}
                            echo Version: ${params.EXCHANGE_ASSET_VERSION}
                        """
                    } catch (Exception e) {
                        echo "Exchange publication failed: ${e.message}"
                        error("Failed to publish to Anypoint Exchange")
                    }
                }
            }
            post {
                success {
                    echo """
                    ========================================
                      EXCHANGE PUBLICATION SUCCESSFUL
                    ========================================
                      Asset: ${params.APP_NAME}
                      Version: ${params.EXCHANGE_ASSET_VERSION}
                      Org ID: ${ANYPOINT_ORG_ID}
                    ========================================
                    """
                }
            }
        }
        
        // ===== STAGE 7: DEPLOY TO CLOUDHUB 2.0 =====
        stage('Deploy to CloudHub 2.0') {
            when {
                expression { params.SKIP_DEPLOY == false }
            }
            steps {
                script {
                    echo "Deploying to CloudHub 2.0 (${params.DEPLOYMENT_ENV})..."
                    
                    try {
                        // Determine Maven profile based on environment
                        def mavenProfile = mapEnvironmentToProfile(params.DEPLOYMENT_ENV)
                        
                        echo "Using Maven profile: ${mavenProfile}"
                        
                        bat """
                            @echo off
                            
                            REM Export credentials for Maven
                            set MAVEN_OPTS=-Xmx1024m
                            
                            "%MAVEN_HOME%/bin/mvn" deploy ^
                                -P${mavenProfile} ^
                                -DmuleDeploy ^
                                -Denv.connectedApp.id=${CONNECTED_APP_ID} ^
                                -Denv.connectedApp.secret=${CONNECTED_APP_SECRET} ^
                                -Ddeployment.app.name=${params.APP_NAME} ^
                                -Denv.cloudHub.replicas=${params.REPLICAS} ^
                                -Denv.cloudHub.vcores=${params.VCORES} ^
                                -DskipTests ^
                                -Dorg.slf4j.simpleLogger.defaultLogLevel=info ^
                                -B
                            
                            echo Deployment to CloudHub 2.0 successful
                        """
                    } catch (Exception e) {
                        echo "Deployment failed: ${e.message}"
                        error("Deployment to CloudHub 2.0 failed")
                    }
                }
            }
            post {
                success {
                    echo """
                    ========================================
                      CLOUDHUB 2.0 DEPLOYMENT SUCCESS
                    ========================================
                      Environment: ${params.DEPLOYMENT_ENV}
                      App Name: ${params.APP_NAME}
                      Replicas: ${params.REPLICAS}
                      vCores: ${params.VCORES}
                    ========================================
                    """
                }
            }
        }
        
        // ===== STAGE 8: VERIFY DEPLOYMENT =====
        stage('Verify Deployment') {
            when {
                expression { params.SKIP_DEPLOY == false }
            }
            steps {
                script {
                    echo "Verifying deployment..."
                    
                    try {
                        bat '''
                            echo Waiting for application to start (60 seconds)...
                            timeout /t 60 /nobreak
                            
                            echo Checking application health...
                            
                            REM Add your health check here
                            REM curl -f https://%APP_NAME%.cloudhub.io/api/health
                            
                            echo Deployment verification complete
                        '''
                    } catch (Exception e) {
                        echo "Verification check failed: ${e.message}"
                        // Don't fail the build on verification failure
                        unstable("Deployment verification failed")
                    }
                }
            }
        }
    }
    
    // ===== POST BUILD ACTIONS =====
    post {
        always {
            script {
                echo "Cleaning up..."
                
                // Archive build artifacts
                archiveArtifacts artifacts: 'target/*.jar', 
                                 allowEmptyArchive: true,
                                 onlyIfSuccessful: false
                
                echo "Cleanup complete"
            }
        }
        
        success {
            script {
                echo "PIPELINE SUCCESSFUL!"
                echo ""
                echo "========================================"
                echo "      DEPLOYMENT SUCCESSFUL"
                echo "========================================"
                echo "Build: ${BUILD_NAME}"
                echo "Environment: ${params.DEPLOYMENT_ENV}"
                echo "App Name: ${params.APP_NAME}"
                echo "Exchange Version: ${params.EXCHANGE_ASSET_VERSION}"
                echo "Duration: ${currentBuild.durationString}"
                echo "========================================"
            }
        }
        
        failure {
            script {
                echo "PIPELINE FAILED!"
                echo ""
                echo "========================================"
                echo "       DEPLOYMENT FAILED"
                echo "========================================"
                echo "Build: ${BUILD_NAME}"
                echo "Environment: ${params.DEPLOYMENT_ENV}"
                echo "Check logs above for details"
                echo "========================================"
            }
        }
        
        unstable {
            script {
                echo "PIPELINE UNSTABLE (Tests/Verification Failed)"
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
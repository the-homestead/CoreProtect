pipeline {
    agent any
    
    tools {
        // Make sure Jenkins has Maven configured with this name
        maven 'Maven' // Adjust this to match your Jenkins Maven installation name
        jdk 'jdk21'        // Adjust this to match your Jenkins JDK 21 installation name
    }
    
    environment {
        // Set Maven options
        MAVEN_OPTS = '-Xmx1024m -Xms512m'
        // Skip downloading sources and javadocs for faster builds
        MAVEN_CONFIG = '-Dmaven.repo.local=.m2/repository'
    }
    
    options {
        // Keep builds for 30 days
        buildDiscarder(logRotator(daysToKeepStr: '30', numToKeepStr: '50'))
        // Timeout after 30 minutes
        timeout(time: 30, unit: 'MINUTES')
        // Add timestamps to console output
        timestamps()
    }
    
    triggers {
        // Poll SCM every 5 minutes for changes (adjust as needed)
        pollSCM('H/5 * * * *')
    }
    
    stages {
        stage('Checkout') {
            steps {
                echo 'Checking out CoreProtect source code...'
                checkout scm
            }
        }
        
        stage('Build Info') {
            steps {
                script {
                    echo "Building CoreProtect on branch: ${env.BRANCH_NAME}"
                    echo "Build number: ${env.BUILD_NUMBER}"
                    echo "Java version check:"
                    bat 'java -version'
                    echo "Maven version check:"
                    bat 'mvn -version'
                }
            }
        }
        
        stage('Clean') {
            steps {
                echo 'Cleaning previous CoreProtect builds...'
                bat 'mvn clean'
            }
        }
        
        stage('Compile') {
            steps {
                echo 'Compiling CoreProtect...'
                bat 'mvn compile'
            }
        }
        
        stage('Test') {
            steps {
                echo 'Running CoreProtect tests...'
                script {
                    try {
                        // CoreProtect has skipTests=true by default, so we override it
                        bat 'mvn test -DskipTests=false'
                    } catch (Exception e) {
                        echo 'Tests failed, but continuing build...'
                        currentBuild.result = 'UNSTABLE'
                    }
                }
            }
            post {
                always {
                    // Publish test results if they exist
                    script {
                        if (fileExists('**/target/surefire-reports/*.xml')) {
                            junit testResultsPattern: '**/target/surefire-reports/*.xml'
                        }
                    }
                }
            }
        }
        
        stage('Package') {
            steps {
                echo 'Packaging CoreProtect plugin with dependencies...'
                script {
                    // Set the branch property based on the current branch
                    def branchProperty = env.BRANCH_NAME == 'master' ? 'release' : 
                                        env.BRANCH_NAME == 'main' ? 'release' : 
                                        env.BRANCH_NAME == 'develop' ? 'development' : 
                                        'development'
                    
                    echo "Setting project.branch to: ${branchProperty}"
                    bat "mvn package -DskipTests -Dproject.branch=${branchProperty}"
                }
                
                // Verify the shaded JAR was created
                script {
                    def jarFiles = bat(script: 'dir target\\CoreProtect-*.jar /b', returnStdout: true).trim()
                    echo "Generated JAR files: ${jarFiles}"
                    
                    // Check if the main JAR exists (not the original)
                    def mainJar = bat(script: 'dir target\\CoreProtect-*.jar /b | findstr /v original', returnStdout: true).trim()
                    if (mainJar) {
                        echo "Final shaded JAR ready: ${mainJar}"
                    } else {
                        error "Failed to create shaded JAR file"
                    }
                }
            }
        }
        
        stage('Install') {
            when {
                anyOf {
                    branch 'master'
                    branch 'main'
                    branch 'develop'
                }
            }
            steps {
                echo 'Installing CoreProtect to local repository...'
                script {
                    // Set the same branch property for consistency
                    def branchProperty = env.BRANCH_NAME == 'master' ? 'release' : 
                                        env.BRANCH_NAME == 'main' ? 'release' : 
                                        env.BRANCH_NAME == 'develop' ? 'development' : 
                                        'development'
                    
                    bat "mvn install -DskipTests -Dproject.branch=${branchProperty}"
                }
            }
        }
    }
    
    post {
        always {
            echo 'Cleaning up CoreProtect workspace...'
            // Archive only the final shaded CoreProtect JAR (ready to deploy)
            archiveArtifacts artifacts: 'target/CoreProtect-*.jar', 
                           fingerprint: true, 
                           allowEmptyArchive: false,
                           excludes: 'target/original-*.jar,target/*-sources.jar,target/*-javadoc.jar'
        }
        
        success {
            echo 'CoreProtect build completed successfully!'
            script {
                if (env.BRANCH_NAME == 'master' || env.BRANCH_NAME == 'main') {
                    echo 'Master/Main branch build succeeded - CoreProtect plugin is ready for deployment'
                }
            }
        }
        
        failure {
            echo 'CoreProtect build failed!'
            // You can add notification steps here (email, Slack, etc.)
        }
        
        unstable {
            echo 'CoreProtect build completed with test failures'
        }
        
        cleanup {
            // Clean up workspace if needed
            deleteDir()
        }
    }
}
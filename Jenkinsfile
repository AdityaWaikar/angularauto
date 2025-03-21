pipeline {
    agent any
    
    // Define environment variables
    environment {
        // Git Repository Information
        GIT_REPO_URL = 'https://github.com/your-username/your-repo.git'
        GIT_BRANCH = 'main'
        GIT_CREDENTIALS_ID = '100'  // ID of credentials stored in Jenkins
        
        // Google Cloud Information
        GCP_PROJECT_ID = 'hardy-clover-447804-t3'
        GCS_BUCKET = 'htmlbucketaditya'
        GCP_CREDENTIALS_ID = '101'
        BUCKET_PATH = 'htmlbucketaditya/'
        
        // File paths for tracking commits
        WORKSPACE = "${WORKSPACE}"
        OLD_COMMIT_FILE = "${WORKSPACE}\\old_commit.txt"
        CURRENT_COMMIT_FILE = "${WORKSPACE}\\current_commit.txt"
    }
    
    // Configure triggers to poll SCM for changes
    triggers {
        pollSCM('H/5 * * * *') // Polls repository every 5 minutes
    }
    
    stages {
        stage('Checkout') {
            steps {
                // Check out the repository using credentials
                checkout([
                    $class: 'GitSCM',
                    branches: [[name: "*/${env.GIT_BRANCH}"]],
                    doGenerateSubmoduleConfigurations: false,
                    extensions: [],
                    submoduleCfg: [],
                    userRemoteConfigs: [[
                        credentialsId: env.GIT_CREDENTIALS_ID,
                        url: env.GIT_REPO_URL
                    ]]
                ])
                
                // Make a record of previously processed files (if any) - Windows batch version
                bat '''
                if exist %OLD_COMMIT_FILE% (
                    copy %OLD_COMMIT_FILE% %OLD_COMMIT_FILE%.bak
                ) else (
                    git rev-parse HEAD > %OLD_COMMIT_FILE% 2> nul
                )
                git rev-parse HEAD > %CURRENT_COMMIT_FILE%
                '''
            }
        }
        
        stage('Identify New Files') {
            steps {
                // Find files that were added or modified since the last build - Windows version
                script {
                    // Read commit hashes
                    env.OLD_COMMIT = bat(script: "type ${env.OLD_COMMIT_FILE}", returnStdout: true).trim()
                    env.CURRENT_COMMIT = bat(script: "type ${env.CURRENT_COMMIT_FILE}", returnStdout: true).trim()
                    
                    // Extract just the hash (removing the command echo from batch output)
                    env.OLD_COMMIT = env.OLD_COMMIT.readLines().drop(1).join("")
                    env.CURRENT_COMMIT = env.CURRENT_COMMIT.readLines().drop(1).join("")
                    
                    echo "Checking for changes between ${env.OLD_COMMIT} and ${env.CURRENT_COMMIT}"
                    
                    // Get list of changed files - Windows version
                    def changedFilesOutput = bat(
                        script: "git diff --name-status ${env.OLD_COMMIT} ${env.CURRENT_COMMIT} | findstr /R \"^A ^M\"",
                        returnStdout: true
                    ).trim()
                    
                    // Process the output to extract just the filenames
                    def lines = changedFilesOutput.readLines().drop(1) // Drop the first line (command echo)
                    def fileList = []
                    
                    lines.each { line ->
                        if (line.trim()) {
                            def parts = line.trim().split("\\s+", 2)
                            if (parts.length > 1) {
                                fileList.add(parts[1])
                            }
                        }
                    }
                    
                    env.CHANGED_FILES = fileList.join("\n")
                    
                    if (env.CHANGED_FILES) {
                        echo "New or modified files detected: ${env.CHANGED_FILES}"
                    } else {
                        echo "No new or modified files detected"
                    }
                }
            }
        }
        
        stage('Authenticate with GCP') {
            when {
                expression { return env.CHANGED_FILES != '' }
            }
            steps {
                // Authenticate with GCP using credentials - Windows version
                withCredentials([file(credentialsId: env.GCP_CREDENTIALS_ID, variable: 'GCP_KEY')]) {
                    bat "gcloud auth activate-service-account --key-file=%GCP_KEY%"
                    bat "gcloud config set project ${GCP_PROJECT_ID}"
                }
            }
        }
        
        stage('Copy to GCS Bucket') {
            when {
                expression { return env.CHANGED_FILES != '' }
            }
            steps {
                script {
                    // Convert the list of files into an array
                    def files = env.CHANGED_FILES.split('\n')
                    
                    // Copy each file to GCS bucket - Windows version
                    files.each { file ->
                        if (file?.trim()) {
                            echo "Copying file: ${file}"
                            bat "gsutil cp ${file} gs://${GCS_BUCKET}/${BUCKET_PATH}/"
                        }
                    }
                }
            }
        }
        
        stage('Cleanup') {
            steps {
                // Save the current commit hash for the next run - Windows version
                bat "copy %CURRENT_COMMIT_FILE% %OLD_COMMIT_FILE% /Y"
            }
        }
    }
    
    post {
        success {
            echo "Files have been successfully copied to GCS bucket"
        }
        failure {
            echo "Failed to copy files to GCS bucket"
            // For debugging on Windows
            bat "dir %WORKSPACE%"
            bat "if exist %OLD_COMMIT_FILE% type %OLD_COMMIT_FILE%"
            bat "if exist %CURRENT_COMMIT_FILE% type %CURRENT_COMMIT_FILE%"
        }
    }
}
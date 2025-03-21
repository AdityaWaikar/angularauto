pipeline {
    agent any
    
    // Define environment variables
    environment {
        // Git Repository Information
        GIT_REPO_URL = 'https://github.com/AdityaWaikar/angularauto.git'
        GIT_BRANCH = 'main'
        GIT_CREDENTIALS_ID = '100'  // ID of credentials stored in Jenkins
        
        // Google Cloud Information
        GCP_PROJECT_ID = 'hardy-clover-447804-t3'
        GCS_BUCKET = 'htmlbucketaditya'
        GCP_CREDENTIALS_ID = '101'
        BUCKET_PATH = ''
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
                
                // Make a record of previously processed files (if any)
               
            }
        }

        stage('Save Current Commit') {
            steps {
                script {
                    // Save the current commit hash to .current_commit
                    bat 'git rev-parse HEAD > .current_commit'
                    echo "Current commit hash saved in .current_commit"
                }
            }
        }

        stage('Initialize Commit History') {
            steps {
                 script {
                    if (!fileExists('.old_commit')) {
                    echo "Initializing .old_commit file for the first run"
                    bat 'git rev-parse HEAD > .old_commit'
                    }
                 }
             }
        }

        
        stage('Identify New Files') {
            steps {
                // Find files that were added or modified since the last build
                script {
                    env.OLD_COMMIT = env.GIT_PREVIOUS_COMMIT ?: 'HEAD~1' // Default to the last commit
                    env.CURRENT_COMMIT = env.GIT_COMMIT
                    
                    echo "Checking for changes between ${env.OLD_COMMIT} and ${env.CURRENT_COMMIT}"
                    
                    // Get list of all files that were changed
                    env.CHANGED_FILES = bat(
                        script: "git diff --name-status ${env.OLD_COMMIT} ${env.CURRENT_COMMIT} | findstr \"A M\"",
                        returnStdout: true
                    ).trim()

                    
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
                // Authenticate with GCP using the configured service account credentials
                withCredentials([file(credentialsId: env.GCP_CREDENTIALS_ID, variable: 'GCP_KEY')]) {
                    bat "gcloud auth activate-service-account --key-file=${GCP_KEY}"
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
                    // Split the list of changed files into an array
                    def files = env.CHANGED_FILES.split('\n')
            
                    // Copy each file to GCS bucket
                    files.each { file ->
                        if (file?.trim()) {
                            echo "Copying file: ${file}"
                    
                            // Construct the destination path dynamically
                            def destinationPath = BUCKET_PATH ? "${BUCKET_PATH}/" : ''
                    
                            // Execute the gsutil copy command
                            bat "gsutil -m cp -r ${file} gs://${GCS_BUCKET}/${destinationPath}"
                        }
                    }
                }
            }
        }


        
        stage('Cleanup') {
            steps {
                // Save the current commit hash for the next run
                bat 'copy .current_commit .previous_commit'
            }
        }
    }
    
    post {
        success {
            echo "Files have been successfully copied to GCS bucket"
        }
        failure {
            echo "Failed to copy files to GCS bucket"
        }
    }
}
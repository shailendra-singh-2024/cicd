pipeline {
    agent any

    environment {
        GIT_CREDENTIALS_ID = 'jenkins-goapptiv-github-app' // Use this ID for GitHub credentials
        REPO_URL = 'https://github.com/shailendra-singh-2024/cicd.git' // Your repository URL
        BRANCH_NAME = 'main' // Replace with your branch name
    }

    stages {
        stage('Checkout') {
            steps {
                script {
                    checkout([$class: 'GitSCM', 
                              branches: [[name: "${BRANCH_NAME}"]],
                              userRemoteConfigs: [[url: "${REPO_URL}", credentialsId: "${GIT_CREDENTIALS_ID}"]]
                    ])
                }
            }
        }
        
        stage('Build') {
            steps {
                sh 'echo Building the project...'
            }
        }
        
        stage('Deploy') {
            steps {
                sh 'echo Deploying the project...'
            }
        }
    }

    post {
        success {
            echo 'Pipeline completed successfully!'
        }
        failure {
            echo 'Pipeline failed!'
        }
    }
}

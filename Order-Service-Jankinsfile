pipeline {
    agent any

    environment {
        STATE_FILE = 'terraform/cod-order-management-service.tfstate'
        GCR_REGISTRY = "asia.gcr.io"
        GCR_PROJECT_ID = "goapptiv"
        GCR_IMAGE_NAME = "cod-order-management-service"
        GIT_CREDENTIALS_ID = 'jenkins-goapptiv-github-app'
        REPO_URL = 'https://github.com/GoApptiv/order-management-service-laravel.git'
        BRANCH_NAME = 'load-balancing-scaling'
        TERRAFORM_REPO_URL = 'https://github.com/GoApptiv/cod-microservices-terraform-config'
        TERRAFORM_BRANCH = 'master'
    }

    stages {
        stage('Checkout Source Code') {
            steps {
                script {
                    echo "Starting Checkout of Source Code"
                    checkout([$class: 'GitSCM',
                              branches: [[name: "${BRANCH_NAME}"]],
                              userRemoteConfigs: [[url: "${REPO_URL}", credentialsId: "${GIT_CREDENTIALS_ID}"]]
                    ])
                    echo "Source Code Checkout Completed Successfully"
                }
            }
        }

        stage('Global GCP Authentication') {
            steps {
                script {
                    echo "Starting GCP Authentication"
                    withCredentials([file(credentialsId: 'proj-goapptiv-jenkins-admin', variable: 'GOOGLE_APPLICATION_CREDENTIALS')]) {
                        sh 'gcloud auth activate-service-account --key-file=$GOOGLE_APPLICATION_CREDENTIALS'
                        sh 'gcloud auth configure-docker ${GCR_REGISTRY}'
                        echo "GCP Authentication Completed Successfully"
                    }
                }
            }
        }

        stage('Write Secret File') {
            steps {
                script {
                    echo "Writing Secret File"
                    withCredentials([file(credentialsId: 'cod-order-management-service-laravel-env', variable: 'SECRET_FILE_PATH')]) {
                        writeFile file: '.env', text: readFile(SECRET_FILE_PATH)
                        echo "Secret File Written Successfully"
                    }

                    def devopsEmails = sh(script: "grep APP_DEVOPS_SUPPORT_EMAILS .env | cut -d '=' -f2", returnStdout: true).trim()
                    env.APP_DEVOPS_SUPPORT_EMAILS = devopsEmails
                    echo "DevOps Support Emails Set: ${devopsEmails}"
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    echo "Starting Docker Image Build"
                    sh 'chmod -R 777 storage'
                    withCredentials([string(credentialsId: 'goapptiv-composer-github-token', variable: 'COMPOSER_TOKEN')]) {
                        sh """
                        docker build --build-arg COMPOSER_TOKEN=${COMPOSER_TOKEN} \\
                                     -t ${GCR_REGISTRY}/${GCR_PROJECT_ID}/${GCR_IMAGE_NAME}:latest \\
                                     -f ./image/php/Dockerfile . --no-cache
                        """
                        echo "Docker Image Build Completed Successfully"
                    }
                }
            }
        }

        stage('Push to GCR') {
            steps {
                script {
                    echo "Starting Docker Image Push to GCR"
                    sh 'docker push ${GCR_REGISTRY}/${GCR_PROJECT_ID}/${GCR_IMAGE_NAME}:latest'
                    echo "Docker Image Pushed to GCR Successfully"
                }
            }
        }

        stage('Checkout Terraform Configuration') {
            steps {
                script {
                    echo "Starting Checkout of Terraform Configuration"
                    checkout([$class: 'GitSCM',
                              branches: [[name: "${TERRAFORM_BRANCH}"]],
                              userRemoteConfigs: [[url: "${TERRAFORM_REPO_URL}", credentialsId: "${GIT_CREDENTIALS_ID}"]]
                    ])
                    echo "Terraform Configuration Checkout Completed Successfully"
                }
            }
        }

        stage('Terraform Apply') {
            environment {
                GOOGLE_APPLICATION_CREDENTIALS = credentials('proj-goapptiv-jenkins-admin')
            }
            agent {
                docker {
                    image 'hashicorp/terraform:latest'
                    args '-v $WORKSPACE:/workspace --entrypoint=""'
                }
            }
            steps {
                script {
                    echo "Starting Terraform Apply"
                    withCredentials([string(credentialsId: 'goapptiv-composer-github-token', variable: 'GITHUB_TOKEN')]) {
                        dir('terraform') { // Ensure this directory matches your Terraform configuration path
                            sh """
                            echo "Initializing Terraform..."
                            terraform init

                            echo "Applying Terraform configuration..."
                            terraform apply -auto-approve -input=false \\
                                            -state=${STATE_FILE} \\
                                            -var "github_token=${GITHUB_TOKEN}" \\
                                            -var "deployment_id=${BUILD_NUMBER}" \\
                                            -target=module.order-service
                            """
                            echo "Terraform Apply Completed Successfully"
                        }
                    }
                }
            }
        }
    }

    post {
        success {
            script {
                echo "Pipeline Completed Successfully"
            }
            mail to: "${env.APP_DEVOPS_SUPPORT_EMAILS}",
                subject: "Pipeline Successful: ${env.JOB_NAME} Build #${env.BUILD_NUMBER}",
                body: "The Jenkins pipeline ${env.JOB_NAME} has completed successfully.\n\n" +
                      "Build Number: ${env.BUILD_NUMBER}\n" +
                      "Duration: ${currentBuild.durationString}\n" +
                      "Status: SUCCESS"
        }

        failure {
            script {
                echo "Pipeline Failed"
            }
            mail to: "${env.APP_DEVOPS_SUPPORT_EMAILS}",
                subject: "Pipeline Failed: ${env.JOB_NAME} Build #${env.BUILD_NUMBER}",
                body: "The Jenkins pipeline ${env.JOB_NAME} has failed. Please check the console output for details.\n\n" +
                      "Build Number: ${env.BUILD_NUMBER}\n" +
                      "Duration: ${currentBuild.durationString}\n" +
                      "Status: FAILURE"
        }

        always {
            script {
                echo "Cleaning Up Workspace and Docker"
                cleanWs()
                sh "docker system prune -af || true"
                echo "Cleanup Completed"
            }
        }
    }
}

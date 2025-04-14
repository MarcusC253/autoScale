pipeline {
    agent any

    environment {
        AWS_REGION = 'us-east-1'
        SONARQUBE_URL = "https://sonarcloud.io"
        JIRA_SITE = "https://derrickweil.atlassian.net"
        JIRA_PROJECT = "SCRUM"
    }

    stages {

        stage('Set AWS Credentials') {
            steps {
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'devops253' 
                ]]) {
                    sh '''
                    echo "AWS_ACCESS_KEY_ID: $AWS_ACCESS_KEY_ID"
                    aws sts get-caller-identity
                    '''
                }
            }
        }

        stage('Checkout Code') {
            steps {
                git branch: 'main', url: 'https://github.com/MarcusC253/autoScale.git'
            }
        }

        stage('Secret Scanning with TruffleHog') {
            steps {
                script {
                    sh '''
                        mkdir -p /home/jenkins/bin

                        if ! command -v trufflehog &> /dev/null
                        then
                            echo 'TruffleHog not found! Installing...'
                            curl -sSfL https://raw.githubusercontent.com/trufflesecurity/trufflehog/main/scripts/install.sh | sh -s -- -b /home/jenkins/bin
                        fi

                        echo 'Running TruffleHog Scan...'
                        PATH="/home/jenkins/bin:$PATH" trufflehog git --entropy=False --only-verified --json . || echo 'No secrets found'
                    '''
                }
            }
        }

        stage('Static Code Analysis (SAST)') {
            steps {
                script {
                    withCredentials([string(credentialsId: 'SONARQUBE_TOKEN', variable: 'SONAR_TOKEN')]) {
                        def scanStatus = sh(script: '''
                            ${SONAR_SCANNER_HOME}/bin/sonar-scanner \
                            -Dsonar.projectKey=derrickSh43_autoScale \
                            -Dsonar.organization=derricksh43 \
                            -Dsonar.host.url=${SONARQUBE_URL} \
                            -Dsonar.login=''' + SONAR_TOKEN, returnStatus: true)

                        if (scanStatus != 0) {
                            echo "Creating Jira ticket for SAST failure..."
                            sh '''
                            curl -X POST -H "Content-Type: application/json" \
                            -u $JIRA_USERNAME:$JIRA_API_TOKEN \
                            --data '{
                              "fields": {
                                "project": { "key": "'"$JIRA_PROJECT"'" },
                                "summary": "Static Code Analysis Failed",
                                "description": "SonarQube scan detected issues in your code.",
                                "issuetype": { "name": "Bug" },
                                "priority": { "name": "High" }
                              }
                            }' \
                            $JIRA_SITE/rest/api/2/issue/
                            '''
                            error("SonarQube found security vulnerabilities!")
                        }
                    }
                }
            }
        }

        stage('Snyk Security Scan') {
            steps {
                script {
                    withCredentials([string(credentialsId: 'SNYK_AUTH_TOKEN', variable: 'SNYK_TOKEN')]) {
                        sh "snyk auth ${SNYK_TOKEN}"
                        sh "snyk monitor || echo 'No supported files found, monitoring skipped.'"
                    }
                }
            }
        }

        stage('Initialize Terraform') {
            steps {
                sh 'terraform init'
            }
        }

        stage('Plan Terraform') {
            steps {
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'devops253'
                ]]) {
                    sh '''
                    export AWS_ACCESS_KEY_ID=$AWS_ACCESS_KEY_ID
                    export AWS_SECRET_ACCESS_KEY=$AWS_SECRET_ACCESS_KEY
                    terraform plan -out=tfplan
                    '''
                }
            }
        }

        stage('Apply Terraform') {
            steps {
                input message: "Approve Terraform Apply?", ok: "Deploy"
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'devops253'
                ]]) {
                    sh '''
                    export AWS_ACCESS_KEY_ID=$AWS_ACCESS_KEY_ID
                    export AWS_SECRET_ACCESS_KEY=$AWS_SECRET_ACCESS_KEY
                    terraform apply -auto-approve tfplan
                    '''
                }
            }
        }

        stage('Terraform Destroy') {
            steps {
                input message: 'Are you sure you want to destroy the infrastructure?', ok: 'Proceed with Destroy'
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'devops253'
                ]]) {
                    sh '''
                    export AWS_ACCESS_KEY_ID=$AWS_ACCESS_KEY_ID
                    export AWS_SECRET_ACCESS_KEY=$AWS_SECRET_ACCESS_KEY
                    terraform destroy -auto-approve
                    '''
                }
            }
        }
    }

    post {
        success {
            echo 'Terraform deployment completed successfully!'
        }
        failure {
            echo 'Terraform deployment failed!'
        }
    }
}

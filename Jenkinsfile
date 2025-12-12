pipeline {
      agent { label 'agent' }

    environment {
        // Update these according to your environment
        DEPLOY_SERVER = "10.0.1.156"            // PRIVATE IP of your App EC2
        DEPLOY_PATH   = "/var/www/myapp"          // Directory on App EC2
        SSH_CRED      = "app-server-ssh"          // Jenkins → Manage Credentials → SSH Username with private key
    }

    triggers {
        // Enables GitHub Webhook for MultiBranch pipeline
        githubPush()
    }

    stages {

        stage('Checkout Code') {
            steps {
                checkout scm
            }
        }

        stage('Build & Test (Maven)') {
            steps {
                sh 'mvn clean test'
            }
        }

        stage('Security Scan - OWASP Dependency Check') {
            steps {
                sh '''
                    dependency-check.sh \
                    --scan . \
                    --format HTML \
                    --project git-jenkins-app \
                    --out dependency-check-report
                '''
            }
            post {
                always {
                    archiveArtifacts artifacts: 'dependency-check-report/*', allowEmptyArchive: true
                }

                failure {
                    echo "Security scan failed!"
                }
            }
        }

        stage('Package Application') {
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }

        stage('Archive Artifact') {
            steps {
                archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
            }
        }

        stage('Deploy to Application Server (main branch only)') {
            when {
                branch 'main'
            }
            steps {

                echo "Deploying to application server..."

                // Copy artifact to App Server
                sshagent([SSH_CRED]) {
                    sh '''
                        scp target/*.jar ec2-user@${DEPLOY_SERVER}:${DEPLOY_PATH}/app.jar

                        ssh ec2-user@${DEPLOY_SERVER} "sudo systemctl restart myapp.service"
                    '''
                }
            }
        }
    }

    post {
        always {
            echo "Pipeline completed!"
        }
    }
}

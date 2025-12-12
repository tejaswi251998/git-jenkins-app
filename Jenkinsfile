
  pipeline {
    agent { label 'agent' }

    environment {
        DEPLOY_SERVER = "10.0.1.156"   // Replace with app server EC2
        DEPLOY_PATH   = "/var/www/myapp"              // Folder on app-server
        SSH_CRED      = "app-server-ssh"              // Jenkins SSH Credential ID
        JAVA_HOME     = "/usr/lib/jvm/java-21"        // Your EC2 agent uses Java 21
    }

    stages {

        /* ---------------------------------------------------
         * 1. CHECKOUT CODE
         * ---------------------------------------------------*/
        stage('Checkout') {
            steps {
                checkout scm
                echo "Checked out branch: ${env.BRANCH_NAME}"
            }
        }

        /* ---------------------------------------------------
         * 2. BUILD & TEST WITH MAVEN (Java 21)
         * ---------------------------------------------------*/
        stage('Build & Test') {
            steps {
                sh '''
                    echo "Using Java version:"
                    java -version

                    mvn clean test
                '''
            }
        }

        /* ---------------------------------------------------
         * 3. SECURITY SCAN USING OWASP DEPENDENCY CHECK
         * ---------------------------------------------------*/
        stage('Security Scan - OWASP') {
            steps {
                sh '''
                    echo "Downloading OWASP Dependency Check..."
                    wget https://github.com/jeremylong/DependencyCheck/releases/download/v10.0.2/dependency-check-10.0.2-release.zip -O dc.zip

                    unzip -o dc.zip -d dc

                    echo "Running Dependency Check scan..."
                    dc/dependency-check/bin/dependency-check.sh \
                        --project git-jenkins-app \
                        --scan . \
                        --format HTML \
                        --out dependency-report
                '''
            }
            post {
                always {
                    archiveArtifacts artifacts: 'dependency-report/*'
                }

                failure {
                    echo "Security Scan Failed — Fix vulnerabilities!"
                }
            }
        }

        /* ---------------------------------------------------
         * 4. PACKAGE ARTIFACT (MAVEN)
         * ---------------------------------------------------*/
        stage('Package') {
            steps {
                sh 'mvn -B -DskipTests clean package'

                archiveArtifacts artifacts: '**/target/*.jar', fingerprint: true
            }
        }

        /* ---------------------------------------------------
         * 5. DEPLOY TO APP SERVER ONLY ON MAIN BRANCH
         * ---------------------------------------------------*/
        stage('Deploy to App Server EC2') {
            when {
                branch 'main'
            }
            steps {
                echo "Deploying to EC2 Instance: ${DEPLOY_SERVER}"

                sshagent(credentials: [SSH_CRED]) {
                    sh '''
                        echo "Transferring JAR to App Server..."
                        scp -o StrictHostKeyChecking=no target/*.jar ec2-user@'${DEPLOY_SERVER}':'${DEPLOY_PATH}/app.jar'

                        echo "Restarting Application Service..."
                        ssh -o StrictHostKeyChecking=no ec2-user@'${DEPLOY_SERVER}' '
                            sudo systemctl stop myapp || true
                            sudo systemctl start myapp
                        '

                        echo "Deployment Successful!"
                    '''
                }
            }
        }
    }

    post {
        always {
            echo "Pipeline completed for branch: ${env.BRANCH_NAME}"
        }
    }
}


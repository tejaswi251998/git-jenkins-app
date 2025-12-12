// Jenkinsfile (Declarative Pipeline)

pipeline {
    agent {label 'agent'}

  
    stages {
        stage('Checkout Code') {
            
            steps {
                // The Multibranch pipeline automatically checks out the correct branch
                checkout scm
            }
        }

        stage('Build & Test using Maven') {
            
            steps {
                // Assumes Maven is installed and accessible in the agent's PATH
                sh 'mvn clean test'
            }
        }

        stage('Security Scan Stage') {
         
            steps {
                // Example using OWASP Dependency Check
                // You must install the OWASP Dependency-Check plugin in Jenkins
                dependencyCheck analysis: [
                    // Configure your project settings here
                    // e.g., suppressionFile: 'suppressions.xml'
                ],
                // Add the failure threshold configuration
                failBuildOnCVSS: 7, // Fail if any vulnerability has a CVSS score >= 7.0
                skipOnScmChange: false,
                severity: 'HIGH', // Only report HIGH severity
                format: 'HTML',
                outputDirectory: 'dependency-check-report'
            }
        }

        stage('Package Stage') {
         
            steps {
                sh 'mvn package -DskipTests' // Build the artifact
                // Archive the resulting .jar or .war file (adjust path as needed)
                archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
            }
        }

        stage('Deploy to Application Server') {
            // The deployment should only happen for the 'main' branch
            when {
                branch 'main'
            }
          
            environment {
                // Environment variables for the deployment script
                APP_SERVER_IP = 'Your_Application_Server_IP' // Replace with your App Server IP
                SSH_CREDENTIAL_ID = 'appserver-ssh-key' // Your SSH credential ID from Step 3
                TARGET_USER = 'ec2-user' // The username for the app server
                TARGET_PATH = '/opt/app' // Where the app will be deployed
                ARTIFACT_NAME = 'target/your-app-name.jar' // REPLACE with your actual artifact name
            }
            steps {
                withCredentials([sshUserPrivateKey(
                    credentialsId: env.SSH_CREDENTIAL_ID,
                    keyFileVariable: 'SSH_KEY',
                    usernameVariable: 'SSH_USER'
                )]) {
                    // 1. Copy artifact to the Application Server using SCP
                    sh "scp -i ${SSH_KEY} ${ARTIFACT_NAME} ${TARGET_USER}@${env.APP_SERVER_IP}:${env.TARGET_PATH}/"

                    // 2. Restart the application on the Application Server using SSH
                    sh '''
                        ssh -i ${SSH_KEY} ${TARGET_USER}@${env.APP_SERVER_IP} "
                            # Stop the existing application process (adjust command as needed)
                            sudo systemctl stop my-app-service.service || true
                            
                            # Start the new application
                            java -jar ${TARGET_PATH}/${ARTIFACT_NAME} &
                        "
                    '''
                }
            }
        }
    }
}

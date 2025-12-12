pipeline {
    agent { label 'agent' }

    tools {
        jdk 'jdk21'          // Jenkins Tool → Manage Jenkins > Tools
        maven 'maven3911'        // Jenkins Tool → Manage Jenkins > Tools
    }

    environment {
        APP_SERVER = "ubuntu@44.193.0.46"
        APP_PATH = "/var/www/myapp"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build') {
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }

        stage('Security Scan - Trivy') {
            steps {
                sh '''
                echo "Running Trivy Scan..."
                trivy fs --exit-code 0 --format table --output trivy-report.txt .
                '''
            }
            post {
                always {
                    archiveArtifacts artifacts: 'trivy-report.txt', fingerprint: true
                }
            }
        }

        stage('Deploy to App Server') {
            steps {
                sh '''
                echo "Deploying build artifact to Ubuntu EC2..."

                # Copy JAR
                scp -o StrictHostKeyChecking=no target/*.j*

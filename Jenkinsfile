pipeline {
    agent { label 'agent' }   // your Jenkins build agent label

    tools {
        jdk 'jdk21'          // Must match Jenkins -> Global Tool Configuration
        maven 'maven3911'        // Must match Jenkins -> Global Tool Configuration
    }

    environment {
        PATH = "/snap/bin:${env.PATH}"     // Required for Trivy installed via snap
        APP_SERVER = "ubuntu@44.193.0.46"   // Replace with your Deployment EC2
        APP_PATH   = "/var/www/myapp"      // Folder where your app runs
    }

    stages {

        /* -------------------------------------------------------------
         * CHECKOUT CODE
         * ------------------------------------------------------------- */
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        /* -------------------------------------------------------------
         * BUILD & TEST
         * ------------------------------------------------------------- */
        stage('Build & Test') {
            steps {
                sh 'mvn clean test'
            }
        }

        /* -------------------------------------------------------------
         * SECURITY SCAN USING TRIVY
         * FAIL IF HIGH OR CRITICAL VULNERABILITIES FOUND
         * ------------------------------------------------------------- */
        stage('Security Scan - Trivy') {
            steps {
                sh '''
                echo "Running Trivy filesystem security scan..."
                trivy fs . --format json --output trivy-report.json

                # Count HIGH and CRITICAL vulns
                HIGH=$(trivy fs . --severity HIGH --exit-code 0 | grep -c HIGH || true)
                CRITICAL=$(trivy fs . --severity CRITICAL --exit-code 0 | grep -c CRITICAL || true)

                echo "HIGH vulnerabilities found: $HIGH"
                echo "CRITICAL vulnerabilities found: $CRITICAL"

                # Fail pipeline if any HIGH or CRITICAL exist
                if [ "$HIGH" -gt 0 ] || [ "$CRITICAL" -gt 0 ]; then
                    echo "❌ Security scan failed: High/Critical vulnerabilities detected!"
                    exit 1
                fi

                echo "✔ Security Scan Passed"
                '''
            }

            post {
                always {
                    echo "Archiving Trivy report..."
                    archiveArtifacts artifacts: 'trivy-report.json', fingerprint: true
                }
            }
        }

        /* -------------------------------------------------------------
         * PACKAGE ARTIFACT
         * ------------------------------------------------------------- */
        stage('Package') {
            steps {
                sh 'mvn clean package -DskipTests'
            }

            post {
                success {
                    archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
                }
            }
        }

        /* -------------------------------------------------------------
         * DEPLOY TO APP SERVER (ONLY MAIN BRANCH)
         * ------------------------------------------------------------- */
        stage('Deploy to App Server') {
            when {
                branch 'main'    // Deploy only for main branch
            }
            steps {
                sh '''
                echo "Deploying application to the App Server..."

                # Copy JAR file to application server
                scp -o StrictHostKeyChecking=no target/*.jar ${APP_SERVER}:${APP_PATH}/app.jar

                # Stop running app (if any) and start new version
                ssh -o StrictHostKeyChecking=no ${APP_SERVER} "
                    pkill -f app.jar || true
                    nohup java -jar ${APP_PATH}/app.jar > ${APP_PATH}/app.log 2>&1 &
                "

                echo "✔ Deployment Completed Successfully"
                '''
            }
        }
    }
}

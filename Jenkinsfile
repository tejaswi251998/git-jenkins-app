pipeline {
    agent { label 'agent' }

    environment {
        MAVEN_OPTS = "-Dmaven.test.failure.ignore=true"
        SEVERITY_THRESHOLD = "HIGH"
        DEPLOY_SERVER = "ubuntu@<EC2-PUBLIC-IP>"
        DEPLOY_PATH = "/opt/app"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build') {
            steps {
                echo "Running: mvn clean test"
                sh 'mvn clean test -Dcheckstyle.skip=true -Dnohttp.skip=true'
            }
            post {
                always {
                    junit 'target/surefire-reports/*.xml'
                }
            }
        }

        stage('Install Trivy (if missing)') {
            steps {
                sh '''
                    if ! command -v trivy &> /dev/null; then
                        echo "Installing Trivy..."
                        sudo apt-get update -y
                        sudo apt-get install wget apt-transport-https gnupg lsb-release -y
                        wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo apt-key add -
                        echo deb https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main | sudo tee /etc/apt/sources.list.d/trivy.list
                        sudo apt-get update -y
                        sudo apt-get install trivy -y
                    else
                        echo "Trivy already installed"
                    fi
                '''
            }
        }

        stage('Security Scan - Trivy') {
            steps {
                sh '''
                    echo "Running Trivy vulnerability scan..."
                    trivy fs --exit-code 0 --format json --output trivy-report.json .

                    HIGH_COUNT=$(trivy fs --severity HIGH --exit-code 0 --format table . | grep -c HIGH || true)
                    echo "High Severity Vulnerabilities Found: $HIGH_COUNT"

                    if [ "$HIGH_COUNT" -gt 0 ]; then
                        echo "❌ Build failed – HIGH vulnerabilities found."
                        exit 1
                    fi
                '''
            }
            post {
                always {
                    archiveArtifacts artifacts: 'trivy-report.json', fingerprint: true
                }
            }
        }

        stage('Deploy to EC2') {
            steps {
                sh '''
                    echo "Deploying artifact to EC2..."

                    ssh -o StrictHostKeyChecking=no ${DEPLOY_SERVER} "mkdir -p ${DEPLOY_PATH}"

                    scp -o StrictHostKeyChecking=no target/*.jar ${DEPLOY_SERVER}:${DEPLOY_PATH}/app.jar

                    ssh -o StrictHostKeyChecking=no ${DEPLOY_SERVER} "
                        pkill -f app.jar || true
                        nohup java -jar ${DEPLOY_PATH}/app.jar > ${DEPLOY_PATH}/app.log 2>&1 &
                    "

                    echo "Deployment Completed Successfully!"
                '''
            }
        }
    }

    post {
        always {
            echo "Pipeline Finished!"
        }
        failure {
            echo "Pipeline Failed ❌"
        }
        success {
            echo "Pipeline Success ✔"
        }
    }
}

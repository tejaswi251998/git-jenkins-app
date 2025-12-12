pipeline {

    agent { label 'agent' }   // your Jenkins agent label

    tools {
        jdk 'jdk21'                 // configure this in Jenkins Tools
        maven 'maven3911'              // configure this in Jenkins Tools
    }

    stages {

        /*-----------------------------------
         1. CHECKOUT SOURCE CODE
        -----------------------------------*/
        stage('Checkout Code') {
            steps {
                checkout scm
            }
        }

        /*-----------------------------------
         2. BUILD AND TEST
        -----------------------------------*/
        stage('Build & Test') {
            steps {
                sh 'mvn clean test'
            }
        }

        /*-----------------------------------
         3. SECURITY SCAN (OWASP Dependency Check)
        -----------------------------------*/
        stage('Security Scan') {
            steps {
                sh '''
                    echo "Downloading OWASP Dependency Check..."

                    wget https://github.com/jeremylong/DependencyCheck/releases/download/v10.0.2/dependency-check-10.0.2-release.zip -O dc.zip
                    
                    unzip dc.zip -d dependency-check

                    echo "Running Security Scan..."
                    dependency-check/bin/dependency-check.sh \
                        --project git-jenkins-app \
                        --scan . \
                        --format HTML \
                        --out dependency-report
                '''
            }
            post {
                always {
                    echo "Archiving Security Reports..."
                    archiveArtifacts artifacts: 'dependency-report/*', allowEmptyArchive: true
                }
            }
        }

        /*-----------------------------------
         4. PACKAGE JAR
        -----------------------------------*/
        stage('Package') {
            steps {
                sh '''
                    echo "Packaging Application..."
                    mvn -B -DskipTests clean package
                '''
            }
            post {
                success {
                    archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
                }
            }
        }

        /*-----------------------------------
         5. DEPLOY ONLY IF BRANCH = main
        -----------------------------------*/
        stage('Deploy to App Server') {
            when {
                branch 'main'
            }
            steps {
                echo "Skipping deploy for non-main branch..."
                echo "Deploying to application server..."

                sshagent(['app-server-ssh']) {
                    sh '''
                        echo "Copying Artifact to Server..."

                        scp -o StrictHostKeyChecking=no target/*.jar ec2-user@54.xx.xx.xx:/home/ec2-user/app.jar

                        echo "Restarting Application..."
                        ssh -o StrictHostKeyChecking=no ec2-user@54.xx.xx.xx "nohup java -jar /home/ec2-user/app.jar > app.log 2>&1 &"
                    '''
                }
            }
        }
    }
}


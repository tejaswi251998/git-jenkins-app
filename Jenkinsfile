pipeline {
  agent { label 'agent' }

  stages {
    stage('Checkout') {
      steps { checkout scm }
    }

    stage('Build') {
      steps { 
        sh 'mvn -B -DskipTests clean package'
        sh 'mvn clean install'
        echo "Building branch: ${env.BRANCH_NAME}"
            }
      
    }
  }
}

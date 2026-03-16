pipeline {
    agent any

    tools {
        maven 'maven'
        jdk 'JDK21'
    }

    environment {
        SONAR_HOST_URL = 'http://192.168.23.135:9000'
    }

    stages {

        stage('Test Credential') {
    steps {
        withCredentials([string(credentialsId: 'snyk-token', variable: 'TOKEN')]) {
            sh 'echo Token loaded'
        }
    }
}
    }

    post {

        always {
            cleanWs()
        }

        success {
            echo "Pipeline completed successfully"
        }

        failure {
            echo "Pipeline failed"
        }
    }
}

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

        stage('Test Snyk CLI') {
    steps {
        sh 'snyk --version'
    }
}
    }

    post {
        always {
            cleanWs()
        }
        success {
            echo "Pipeline completed - only affected services were built."
        }
        failure {
            echo "Pipeline failed. Check logs above."
        }
    }
}

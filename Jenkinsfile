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

        stage('Debug Workspace') {
    steps {
        sh 'pwd'
        sh 'ls -la'
        sh 'ls -la cart || true'
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

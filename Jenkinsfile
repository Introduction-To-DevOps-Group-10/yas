pipeline {
    agent any
    stages {
        stage('Debug workspace') {
            steps {
                sh 'pwd'
                sh 'ls -la'
                sh 'ls cart/ || echo "cart dir not found"'
                sh 'find . -name "mvnw" 2>/dev/null || echo "no mvnw found anywhere"'
                sh 'find . -name "pom.xml" -maxdepth 2 2>/dev/null'
            }
        }

        stage('Security Scan: Snyk - cart only') {
            steps {
                script {
                    // Install root POM dùng đường dẫn tuyệt đối
                    sh """
                        cd ${WORKSPACE}
                        chmod +x mvnw || true
                        ${WORKSPACE}/mvnw install -N -DskipTests || mvn install -N -DskipTests
                    """

                    snykSecurity(
                        snykInstallation: 'snyk',
                        snykTokenId: 'snyk-token',
                        failOnIssues: false,
                        projectName: 'yas-cart',
                        targetFile: 'cart/pom.xml',
                        additionalArguments: '--severity-threshold=high -d'
                    )
                }
            }
        }
    }
    post {
        success { echo "Snyk scan completed" }
        failure { echo "Pipeline failed" }
    }
}

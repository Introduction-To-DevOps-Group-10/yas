pipeline {
    agent any
    stages {
        stage('Security Scan: Snyk - cart only') {
            steps {
                // 1. Install root POM
                sh 'mvn install -N -DskipTests'

                // 2. Install common-library (không có mvnw, dùng mvn hệ thống)
                sh 'mvn install -DskipTests -f common-library/pom.xml'

                // 3. Chạy Snyk scan cho cart
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
    post {
        success { echo "Snyk scan completed" }
        failure { echo "Pipeline failed" }
    }
}

pipeline {
    agent any
    stages {
        stage('Security Scan: Snyk - cart only') {
            steps {
                // Xóa cache Maven bị corrupt từ lần trước
                sh 'rm -rf /var/lib/jenkins/.m2/repository/com/yas'

                // Install root POM
                sh 'mvn install -N -DskipTests'

                // Install common-library vào local repo
                sh 'mvn install -DskipTests -f common-library/pom.xml'

                // Scan cart
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

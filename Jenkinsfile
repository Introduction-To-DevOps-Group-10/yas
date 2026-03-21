pipeline {
    agent any
    stages {
        stage('Security Scan: Snyk') {
            steps {
                // Bước 1: Fix permission mvnw
                sh 'find . -name "mvnw" -exec chmod +x {} +'

                // Bước 2: Install root POM vào local Maven repo
                // để các module con resolve được ${revision} và common-library
                sh './mvnw install -N -DskipTests'

                script {
                    def services = [
                        'cart','customer','order','product','rating',
                        'inventory','media','tax','location','promotion'
                    ]
                    services.each { svc ->
                        if (!fileExists("${svc}/pom.xml")) {
                            echo "Skipping ${svc} (no pom.xml)"
                            return
                        }
                        echo "Running Snyk scan for ${svc}"
                        snykSecurity(
                            snykInstallation: 'snyk',
                            snykTokenId: 'snyk-token',
                            failOnIssues: false,
                            projectName: "yas-${svc}",
                            targetFile: "${svc}/pom.xml",
                            additionalArguments: "--severity-threshold=high -d"
                        )
                    }
                }
            }
        }
    }
    post {
        success { echo "Snyk scan completed" }
        failure { echo "Pipeline failed" }
    }
}

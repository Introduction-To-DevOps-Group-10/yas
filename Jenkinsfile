pipeline {
    agent any

    environment {
        SNYK_TOKEN = credentials('snyk-token')
    }

    stages {

         // ───────────────── SNYK (PLUGIN) ─────────────────
        stage('Security Scan: Snyk') {
            steps {
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
        success {
            echo "Snyk scan completed (debug mode)"
        }
        failure {
            echo "Pipeline failed"
        }
    }
}

pipeline {
    agent any

    tools {
        maven 'maven'
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Snyk Scan (Full Project)') {
            steps {
                script {

                    echo "Scanning entire project with Snyk..."

                    snykSecurity(
                        snykInstallation: 'snyk',
                        snykTokenId: 'snyk-token',
                        failOnIssues: false, // chỉ scan, không fail
                        projectName: "full-project-scan",
                        targetFile: "pom.xml", // root pom
                        additionalArguments: """
                            --all-projects \
                            --severity-threshold=low \
                            --detection-depth=4 \
                            -d
                        """
                    )
                }
            }
        }
    }

    post {
        success {
            echo "Snyk full scan completed"
        }
        failure {
            echo "Snyk scan failed"
        }
    }
}

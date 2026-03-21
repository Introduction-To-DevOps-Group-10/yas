pipeline {
    agent any

    environment {
        REVISION = "1.0-SNAPSHOT"
        MAVEN_REPO = "/var/lib/jenkins/.m2/repository"
    }

    stages {
        stage('Prepare Maven') {
            steps {
                // Xóa cache cũ
                sh 'rm -rf ${MAVEN_REPO}/com/yas'

                // Install root POM
                sh 'mvn install -N -DskipTests -Drevision=${REVISION} -Dmaven.repo.local=${MAVEN_REPO}'

                // Install common-library với flatten để resolve ${revision}
                sh 'mvn flatten:flatten install -DskipTests -Drevision=${REVISION} -Dmaven.repo.local=${MAVEN_REPO} -f common-library/pom.xml'
            }
        }

        stage('Security Scan: Snyk') {
            steps {
                script {
                    def services = [
                        'cart', 'customer', 'order', 'product', 'rating',
                        'inventory', 'media', 'tax', 'location', 'promotion'
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
                            additionalArguments: '--severity-threshold=high -d'
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

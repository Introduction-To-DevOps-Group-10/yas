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
                sh 'rm -rf /var/lib/jenkins/.m2/repository/com/yas'

                // Install root POM
                sh 'mvn install -N -DskipTests -Drevision=${REVISION} -Dmaven.repo.local=${MAVEN_REPO}'

                // Install common-library với flatten plugin để resolve ${revision} trong pom
                sh 'mvn flatten:flatten install -DskipTests -Drevision=${REVISION} -Dmaven.repo.local=${MAVEN_REPO} -f common-library/pom.xml'

                // Verify cart resolve được
                sh 'mvn dependency:resolve -Drevision=${REVISION} -Dmaven.repo.local=${MAVEN_REPO} -f cart/pom.xml'
            }
        }

        stage('Security Scan: Snyk - cart only') {
            steps {
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

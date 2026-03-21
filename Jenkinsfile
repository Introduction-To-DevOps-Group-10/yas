pipeline {
    agent any

    environment {
        // Buộc tất cả Maven process (kể cả của Snyk) dùng cùng 1 local repo
        MAVEN_OPTS = "-Dmaven.repo.local=/var/lib/jenkins/.m2/repository"
    }

    stages {
        stage('Prepare Maven') {
            steps {
                // Xóa cache lỗi cũ
                sh 'rm -rf /var/lib/jenkins/.m2/repository/com/yas'

                // Install root POM với revision rõ ràng
                sh 'mvn install -N -DskipTests -Drevision=1.0-SNAPSHOT'

                // Install common-library
                sh 'mvn install -DskipTests -Drevision=1.0-SNAPSHOT -f common-library/pom.xml'

                // Verify cart có thể resolve dependency (test trước khi Snyk chạy)
                sh 'mvn dependency:resolve -Drevision=1.0-SNAPSHOT -f cart/pom.xml -o || mvn dependency:resolve -Drevision=1.0-SNAPSHOT -f cart/pom.xml'
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
                    additionalArguments: '--severity-threshold=high -d --maven-aggregate-project'
                )
            }
        }
    }

    post {
        success { echo "Snyk scan completed" }
        failure { echo "Pipeline failed" }
    }
}

pipeline {
    agent any

    environment {
        REVISION = "1.0-SNAPSHOT"
        MAVEN_REPO = "/var/lib/jenkins/.m2/repository"
    }

    stages {
        stage('Prepare Maven') {
            steps {
                sh 'rm -rf ${MAVEN_REPO}/com/yas'
                sh 'mvn install -N -DskipTests -Drevision=${REVISION} -Dmaven.repo.local=${MAVEN_REPO}'
                sh 'mvn flatten:flatten install -DskipTests -Drevision=${REVISION} -Dmaven.repo.local=${MAVEN_REPO} -f common-library/pom.xml'
            }
        }

        stage('Security Scan: Snyk - cart') {
            steps {
                snykSecurity(
                    snykInstallation: 'snyk',
                    snykTokenId: 'snyk-token',
                    // Đổi thành true để pipeline FAIL khi có vulnerability
                    failOnIssues: true,
                    projectName: 'yas-cart',
                    targetFile: 'cart/pom.xml',
                    // Đổi thành low để bắt tất cả mức độ
                    additionalArguments: '--severity-threshold=low -d'
                )
            }
        }
    }

    post {
        always {
            // Hiển thị report JSON để đọc kết quả
            sh '''
                REPORT=$(ls -t *snyk_report.json 2>/dev/null | head -1)
                if [ -n "$REPORT" ]; then
                    echo "=== SNYK REPORT ==="
                    cat "$REPORT" | python3 -m json.tool 2>/dev/null || cat "$REPORT"
                fi
            '''
        }
        success { echo "No vulnerabilities found" }
        failure { echo "Vulnerabilities detected!" }
    }
}

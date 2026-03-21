pipeline {
    agent any

    environment {
        REVISION   = "1.0-SNAPSHOT"
        MAVEN_REPO = "/var/lib/jenkins/.m2/repository"
    }

    stages {

        stage('Prepare Maven') {
            steps {
                sh 'rm -rf ${MAVEN_REPO}/com/yas'
                sh '''
                    mvn install -N \
                        -DskipTests \
                        -Drevision=${REVISION} \
                        -Dmaven.repo.local=${MAVEN_REPO}
                '''
                sh '''
                    mvn flatten:flatten install \
                        -DskipTests \
                        -Drevision=${REVISION} \
                        -Dmaven.repo.local=${MAVEN_REPO} \
                        -f common-library/pom.xml
                '''
            }
        }

        stage('Security Scan: Snyk - cart') {
            steps {
                // snykSecurity plugin tự xử lý được DefaultSnykApiToken
                // Chỉ dùng plugin để lấy token, còn output thì capture qua sh
                script {
                    // Dùng snykSecurity với additionalArguments không có --json
                    // để output là human-readable text thẳng vào log
                    snykSecurity(
                        snykInstallation    : 'snyk',
                        snykTokenId         : 'snyk-token',
                        failOnIssues        : false,
                        monitorProjectOnBuild: false,   // tắt monitor để gọn log
                        projectName         : 'yas-cart',
                        targetFile          : 'cart/pom.xml',
                        // Không dùng --json → Snyk in text ra thẳng log
                        additionalArguments : '--severity-threshold=high',
                        
                    )
                }
            }
        }

    }

    post {
        success { echo "✅ Pipeline hoàn thành." }
        failure { echo "❌ Pipeline thất bại." }
    }
}

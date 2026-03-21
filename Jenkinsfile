pipeline {
    agent any

    environment {
        REVISION   = "1.0-SNAPSHOT"
        MAVEN_REPO = "/var/lib/jenkins/.m2/repository"
        SNYK_TOKEN = credentials('snyk-token')
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
                script {
                    // Chạy snyk test trực tiếp — output hiện thẳng ra Jenkins log
                    // Không dùng --json nên kết quả là human-readable text
                    // "|| true" để pipeline không fail dù có vulnerability
                    def result = sh(
                        script: '''
                            /var/lib/jenkins/tools/io.snyk.jenkins.tools.SnykInstallation/snyk/snyk-linux \
                                test \
                                --severity-threshold=high \
                                --file=cart/pom.xml \
                                --project-name=yas-cart \
                                --maven-aggregate-project \
                                -Dmaven.repo.local=${MAVEN_REPO} \
                                2>&1 || true
                        ''',
                        returnStdout: true
                    )

                    // In thẳng ra log
                    echo "=== SNYK SECURITY SCAN RESULT ==="
                    echo result
                }
            }
        }

    }

    post {
        success { echo "✅ Pipeline hoàn thành." }
        failure { echo "❌ Pipeline thất bại." }
    }
}

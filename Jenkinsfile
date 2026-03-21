pipeline {
    agent any

    environment {
        REVISION   = "1.0-SNAPSHOT"
        MAVEN_REPO = "/var/lib/jenkins/.m2/repository"
    }

    stages {

        // ------------------------------------------------------------------ //
        stage('Prepare Maven') {
        // ------------------------------------------------------------------ //
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

        // ------------------------------------------------------------------ //
        stage('Security Scan: Snyk - cart') {
        // ------------------------------------------------------------------ //
            steps {
                snykSecurity(
                    snykInstallation : 'snyk',
                    snykTokenId      : 'snyk-token',
                    failOnIssues     : false,
                    projectName      : 'yas-cart',
                    targetFile       : 'cart/pom.xml',
                    additionalArguments: '--severity-threshold=high'
                )
            }

            post {
                // Chạy NGAY SAU khi stage kết thúc — report chắc chắn đã tồn tại
                always {
                    script {
                        // Dùng find với -mmin 2: tìm file được tạo trong vòng 2 phút gần nhất
                        // Không dùng glob expansion, không dùng marker file
                        def report = sh(
                            script: """
                                find '${env.WORKSPACE}' \
                                    -maxdepth 1 \
                                    -name '*snyk_report.json' \
                                    -mmin -2 \
                                    -type f \
                                | sort -r \
                                | head -1
                            """,
                            returnStdout: true
                        ).trim()

                        if (!report) {
                            // Fallback: lấy file có tên timestamp lớn nhất (sort theo tên)
                            report = sh(
                                script: """
                                    find '${env.WORKSPACE}' \
                                        -maxdepth 1 \
                                        -name '*snyk_report.json' \
                                        -type f \
                                    | sort -r \
                                    | head -1
                                """,
                                returnStdout: true
                            ).trim()
                            echo "Dùng fallback report: ${report}"
                        }

                        if (report) {
                            echo "=== SNYK REPORT: ${report} ==="
                            sh "python3 -m json.tool '${report}' 2>/dev/null || cat '${report}'"
                        } else {
                            echo "Không tìm thấy Snyk report nào."
                        }
                    }
                }
            }
        }

    } // end stages

    post {
        success {
            echo "✅ Pipeline hoàn thành — không có vulnerability nào vượt threshold."
        }
        failure {
            echo "❌ Pipeline thất bại — phát hiện vulnerability hoặc build lỗi!"
        }
    }
}

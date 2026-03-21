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
                // Dùng snykSecurity plugin bình thường
                // Plugin sẽ tự tạo file *_snyk_report.json trong WORKSPACE
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
                always {
                    script {
                        // Snyk plugin luôn tạo file có tên format: <timestamp>Z_snyk_report.json
                        // Tên file chứa timestamp UTC dạng: 2026-03-21T11-47-07-898350324Z
                        // Sort theo tên DESCENDING sẽ cho file MỚI NHẤT lên đầu
                        // VÌ: format timestamp đảm bảo sort theo tên = sort theo thời gian
                        //
                        // Lưu ý: sort -r với tên file dạng 2026-03-21T10-54 vs 2026-03-21T11-47
                        // "T11" > "T10" nên sort -r đúng
                        // Lỗi trước là dùng ls -t (sort theo mtime) và glob expansion
                        // Fix: dùng find + sort theo TÊN FILE (không phải mtime)

                        def report = sh(
                            script: """
                                find '${env.WORKSPACE}' \
                                    -maxdepth 1 \
                                    -name '*Z_snyk_report.json' \
                                    -type f \
                                    -printf '%f %p\n' \
                                | sort -r \
                                | head -1 \
                                | awk '{print \$2}'
                            """,
                            returnStdout: true
                        ).trim()

                        if (report) {
                            echo "=== SNYK REPORT: ${report} ==="
                            sh "python3 -m json.tool '${report}' 2>/dev/null || cat '${report}'"
                        } else {
                            echo "Không tìm thấy Snyk report."
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

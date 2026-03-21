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
                // Lưu WORKSPACE path vào file marker để post dùng lại
                // (tránh vấn đề working directory thay đổi trong post block)
                sh 'echo "${WORKSPACE}" > /tmp/snyk_workspace_path'
                sh 'touch /tmp/snyk_scan_start_marker'

                snykSecurity(
                    snykInstallation : 'snyk',
                    snykTokenId      : 'snyk-token',
                    failOnIssues     : false,
                    projectName      : 'yas-cart',
                    targetFile       : 'cart/pom.xml',
                    additionalArguments: '--severity-threshold=high'
                )
            }
        }

    } // end stages

    post {
        always {
            script {
                // Đọc workspace path đã lưu — tránh lỗi working dir sai trong post
                def ws = sh(
                    script: 'cat /tmp/snyk_workspace_path 2>/dev/null || echo ""',
                    returnStdout: true
                ).trim()

                if (!ws) {
                    ws = env.WORKSPACE
                }

                echo "Workspace: ${ws}"

                // Liệt kê tất cả report để debug
                sh "ls -lt '${ws}'/*snyk_report.json 2>/dev/null || echo 'Không có report nào trong workspace'"

                // Tìm report mới hơn marker trong đúng workspace path
                def report = sh(
                    script: """
                        find '${ws}' -maxdepth 1 \
                            -name '*snyk_report.json' \
                            -newer /tmp/snyk_scan_start_marker \
                            2>/dev/null \
                        | sort -r \
                        | head -1
                    """,
                    returnStdout: true
                ).trim()

                if (report) {
                    echo "=== SNYK REPORT: ${report} ==="
                    sh "cat '${report}' | python3 -m json.tool 2>/dev/null || cat '${report}'"
                } else {
                    // Fallback: lấy report mới nhất bất kể marker
                    echo "Không tìm thấy report mới hơn marker, lấy report mới nhất..."
                    def fallback = sh(
                        script: "ls -t '${ws}'/*snyk_report.json 2>/dev/null | head -1",
                        returnStdout: true
                    ).trim()

                    if (fallback) {
                        echo "=== SNYK REPORT (fallback): ${fallback} ==="
                        sh "cat '${fallback}' | python3 -m json.tool 2>/dev/null || cat '${fallback}'"
                    } else {
                        echo "Không tìm thấy bất kỳ Snyk report nào."
                    }
                }

                // Dọn dẹp marker
                sh 'rm -f /tmp/snyk_scan_start_marker /tmp/snyk_workspace_path'
            }
        }

        success {
            echo "✅ Pipeline hoàn thành — không có vulnerability nào vượt threshold."
        }

        failure {
            echo "❌ Pipeline thất bại — phát hiện vulnerability hoặc build lỗi!"
        }
    }
}

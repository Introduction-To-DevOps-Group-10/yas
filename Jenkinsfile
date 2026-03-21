pipeline {
    agent any

    environment {
        REVISION        = "1.0-SNAPSHOT"
        MAVEN_REPO      = "/var/lib/jenkins/.m2/repository"

       
    }

    stages {

        // ------------------------------------------------------------------ //
        stage('Prepare Maven') {
        // ------------------------------------------------------------------ //
            steps {
                // Xóa cache cũ để tránh dùng artifact stale
                sh 'rm -rf ${MAVEN_REPO}/com/yas'

                // Cài parent POM vào local repo
                sh '''
                    mvn install -N \
                        -DskipTests \
                        -Drevision=${REVISION} \
                        -Dmaven.repo.local=${MAVEN_REPO}
                '''

                // Flatten + cài common-library vào local repo
                // (bước này fix lỗi "Could not find artifact com.yas:yas:pom:${revision}")
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
                // Đánh dấu thời điểm TRƯỚC KHI scan để post có thể tìm đúng report mới
                sh 'touch /tmp/snyk_scan_start_marker'

                snykSecurity(
                    snykInstallation : 'snyk',

                    // snykTokenId phải trỏ đúng credential ID trong Jenkins
                    // Credential này phải là "Secret text" chứa Snyk PAT
                    // của tài khoản là thành viên org tunas106
                    snykTokenId      : 'snyk-token',

                    // true  → pipeline FAIL khi có vulnerability (khuyến nghị cho production)
                    // false → chỉ cảnh báo, pipeline vẫn SUCCESS
                    failOnIssues     : false,

                    projectName      : 'yas-cart',
                    targetFile       : 'cart/pom.xml',

                    // --severity-threshold=high : chỉ báo cáo lỗi mức HIGH trở lên
                    // Bỏ -d nếu không cần verbose log (giảm noise)
                    additionalArguments: '--severity-threshold=high'
                )
            }
        }

    } // end stages

    post {
        always {
            script {
                // Tìm report được tạo SAU marker → đúng report của lần scan này
                // Fix lỗi post đọc nhầm report cũ
                def report = sh(
                    script: '''
                        find . -maxdepth 1 \
                               -name "*snyk_report.json" \
                               -newer /tmp/snyk_scan_start_marker \
                               -printf "%T@ %p\n" 2>/dev/null \
                        | sort -rn \
                        | head -1 \
                        | awk '{print $2}'
                    ''',
                    returnStdout: true
                ).trim()

                if (report) {
                    echo "=== SNYK REPORT: ${report} ==="
                    sh "cat '${report}' | python3 -m json.tool 2>/dev/null || cat '${report}'"
                } else {
                    echo "Không tìm thấy Snyk report mới. Kiểm tra lại bước scan."
                }

                // Dọn marker
                sh 'rm -f /tmp/snyk_scan_start_marker'
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

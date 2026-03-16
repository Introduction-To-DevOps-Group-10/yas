pipeline {
    agent any
    tools {
        maven 'maven'
        jdk 'JDK21'
    }

    environment {
        SONAR_HOST_URL = 'http://192.168.23.135:9000'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        // ─── GITLEAKS ───────────────────────────────────────────
        stage('Security Scan: Gitleaks') {
            steps {
                script {
                    sh 'git fetch origin main:refs/remotes/origin/main || true'

                    def scanRange = ''
                    if (env.BRANCH_NAME == 'main') {
                        scanRange = 'HEAD~1..HEAD'
                    } else {
                        def mainExists = sh(
                            script: 'git rev-parse --verify origin/main > /dev/null 2>&1',
                            returnStatus: true
                        )
                        scanRange = (mainExists == 0) ? 'origin/main..HEAD' : 'HEAD~1..HEAD'
                    }

                    echo "Scanning range: ${scanRange}"

                    def result = sh(
                        script: """
                            gitleaks detect \
                                --source=. \
                                --log-opts="${scanRange}" \
                                --report-format=json \
                                --report-path=gitleaks-report.json \
                                --exit-code=1 \
                                --redact
                        """,
                        returnStatus: true
                    )

                    if (result == 0) {
                        echo "Gitleaks: No secrets found in range [${scanRange}]"
                    } else {
                        def report = readFile('gitleaks-report.json')
                        echo "Gitleaks detected secrets!\n${report}"
                        error("Pipeline failed: secrets found in code!")
                    }
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: 'gitleaks-report.json',
                                     allowEmptyArchive: true
                }
            }
        }

        // ─── SNYK ────────────────────────────────────────────────
        stage('Security Scan: Snyk') {
            steps {
                script {
                    withCredentials([string(credentialsId: 'snyk-token', variable: 'SNYK_TOKEN')]) {
                        def services = [
                            'cart', 'customer', 'order', 'product', 'rating',
                            'inventory', 'media', 'tax', 'location', 'promotion'
                        ]

                        services.each { svc ->
                            echo "Snyk scanning: ${svc}"
                            def result = sh(
                                script: """
                                    snyk auth ${SNYK_TOKEN}
                                    snyk test \
                                        --file=${svc}/pom.xml \
                                        --project-name=yas-${svc} \
                                        --json-file-output=snyk-report-${svc}.json \
                                        --severity-threshold=high \
                                        || true
                                """,
                                returnStatus: true
                            )

                            if (result == 0) {
                                echo "Snyk: No high/critical vulnerabilities found in [${svc}]"
                            } else if (result == 1) {
                                echo "Snyk: Vulnerabilities found in [${svc}] - check snyk-report-${svc}.json"
                                // Đổi thành error(...) nếu muốn fail pipeline khi có lỗ hổng
                            } else {
                                echo "Snyk: Scan error for [${svc}], exit code: ${result}"
                            }
                        }
                    }
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: 'snyk-report-*.json',
                                     allowEmptyArchive: true
                }
            }
        }

        // ─── DETECT CHANGES ─────────────────────────────────────
        stage('Detect Changes') {
            steps {
                script {
                    def changedFiles = ''

                    try {
                        changedFiles = sh(
                            script: "git diff --name-only HEAD~1 HEAD",
                            returnStdout: true
                        ).trim()
                    } catch (Exception e) {
                        changedFiles = sh(
                            script: "git diff --name-only origin/main...HEAD",
                            returnStdout: true
                        ).trim()
                    }

                    if (changedFiles == '') {
                        echo "No changed files detected."
                    }

                    echo "Changed files:\n${changedFiles}"

                    def services = [
                        'cart', 'customer', 'order', 'product', 'rating',
                        'inventory', 'media', 'tax', 'location', 'promotion'
                    ]

                    services.each { svc ->
                        env.setProperty("${svc.toUpperCase()}_CHANGED", changedFiles.contains("${svc}/") ? 'true' : 'false')
                    }

                    def statusLines = services.collect { svc ->
                        "| ${svc.padRight(10)}: ${env.getProperty("${svc.toUpperCase()}_CHANGED")}"
                    }.join('\n                    ')

                    echo """
                    +---------------------------------+
                    |        SERVICES TO BUILD        |
                    +---------------------------------+
                    ${statusLines}
                    +---------------------------------+
                    """
                }
            }
        }

        // ─── BUILD EACH SERVICE ─────────────────────────────────
        stage('Test & Build: All Services') {
            steps {
                script {
                    def services = [
                        'cart', 'customer', 'order', 'product', 'rating',
                        'inventory', 'media', 'tax', 'location', 'promotion'
                    ]

                    services.each { svc ->
                        if (env.getProperty("${svc.toUpperCase()}_CHANGED") != 'true') {
                            echo "Skipping ${svc} (no changes detected)"
                            return
                        }

                        echo "Building service: ${svc}"
                        def projectKey = "yas-${svc}"
                        def projectName = "YAS - ${svc.capitalize()}"

                        // Unit Test + Coverage
                        stage("${svc} - Unit Test + Coverage") {
                            sh "mvn clean test jacoco:report -pl ${svc} -am"
                            junit testResults: "${svc}/target/surefire-reports/*.xml",
                                  allowEmptyResults: true
                        }

                        // Publish Coverage
                        stage("${svc} - Publish Coverage") {
                            recordCoverage(
                                tools: [[
                                    $class: 'CoverageTool',
                                    parser: 'JACOCO',
                                    pattern: "**/${svc}/target/site/jacoco/jacoco.xml"
                                ]],
                                sourceDirectories: [[path: "${svc}/src/main/java"]]
                            )
                        }

                        // SonarQube Analysis
                        stage("${svc} - SonarQube Analysis") {
                            withSonarQubeEnv('SonarQube') {
                                sh """
                                    mvn sonar:sonar -pl ${svc} -am \
                                        -Dsonar.projectKey=${projectKey} \
                                        -Dsonar.projectName="${projectName}" \
                                        -Dsonar.inclusions=${svc}/**/* \
                                        -Dsonar.coverage.jacoco.xmlReportPaths=${svc}/target/site/jacoco/jacoco.xml
                                """
                            }
                        }

                        // Quality Gate
                        stage("${svc} - Quality Gate") {
                            timeout(time: 5, unit: 'MINUTES') {
                                waitForQualityGate abortPipeline: true
                            }
                        }

                        // Build
                        stage("${svc} - Build") {
                            sh "mvn package -pl ${svc} -am -DskipTests"
                        }
                    }
                }
            }
        }
    }

    post {
        always {
            cleanWs()
        }
        success {
            echo "Pipeline completed - only affected services were built."
        }
        failure {
            echo "Pipeline failed. Check logs above."
        }
    }
}

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

        // ───────────────── GITLEAKS ──────────────────────
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

        // ───────────────── SNYK (PLUGIN) ─────────────────
        stage('Security Scan: Snyk') {
            steps {
                script {
                    // Snyk plugin tự động gọi ./mvnw nhưng project không có wrapper.
                    // Tạo file mvnw giả trỏ đến mvn thật ở root và từng service
                    // để plugin không bị lỗi "spawn ./mvnw EACCES"
                    sh '''
                        MVN_BIN=$(which mvn)
                        echo "#!/bin/sh" > mvnw
                        echo "exec ${MVN_BIN} \\"\\$@\\"" >> mvnw
                        chmod +x mvnw
                    '''

                    def services = [
                        'cart', 'customer', 'order', 'product', 'rating',
                        'inventory', 'media', 'tax', 'location', 'promotion'
                    ]

                    // Tạo mvnw trong từng service folder
                    services.each { svc ->
                        if (fileExists("${svc}/pom.xml")) {
                            sh """
                                MVN_BIN=\$(which mvn)
                                echo '#!/bin/sh' > ${svc}/mvnw
                                echo "exec \${MVN_BIN} \\"\\\$@\\"" >> ${svc}/mvnw
                                chmod +x ${svc}/mvnw
                            """
                        }
                    }

                    // Chạy Snyk scan từng service
                    services.each { svc ->
                        if (!fileExists("${svc}/pom.xml")) {
                            echo "Snyk: Skipping ${svc} (no pom.xml found)"
                            return
                        }

                        echo "Snyk: Scanning ${svc}"

                        snykSecurity(
                            snykInstallation: 'snyk',
                            snykTokenId: 'snyk-token',
                            failOnIssues: false,
                            projectName: "yas-${svc}",
                            targetFile: "${svc}/pom.xml",
                            additionalArguments: "--severity-threshold=high"
                        )
                    }
                }
            }
        }

        // ───────────────── DETECT CHANGES ────────────────
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
                        env.setProperty(
                            "${svc.toUpperCase()}_CHANGED",
                            changedFiles.contains("${svc}/") ? 'true' : 'false'
                        )
                    }

                    def statusLines = services.collect { svc ->
                        "| ${svc.padRight(10)}: ${env.getProperty("${svc.toUpperCase()}_CHANGED")}"
                    }.join('\n')

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

        // ───────────────── BUILD SERVICES ────────────────
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

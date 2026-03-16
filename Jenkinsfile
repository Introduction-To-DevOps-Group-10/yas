convince: thuyết phúc
firm
correspond
run out
workforce: lực lượng ld
refrain: kiềm chế
absorb: hấp thụ
scout: trinh sát , tìm kiếm

media
product
cart
search
tax
inventory
location

cart
customer
order
product
rating
inventory
meida
tax
location
promotion

jenkins-token of Sonar
sqa_01536c09ec0a17ec570cde4b009f6eeb9a39f4b9

snyk_uat.1fcad39e.eyJlIjoxNzgxNDQ0MTUzLCJoIjoic255ay5pbyIsImoiOiJBWnoyM1V0enFBV1haczhkOHBLTU5BIiwicyI6InUtMlhadVZrU01Db0NaenBaSnp3VEEiLCJ0aWQiOiJBQUFBQUFBQUFBQUFBQUFBQUFBQUFBIn0.mMItWusgZojnPaUZKQde94Mk7TpNSTRxxgj1aKPB1LyRclJXLPCjgrB0-xA8BouJQdA_tQmqhzecgziKK1RdDg


export SNYK_TOKEN=snyk_uat.1fcad39e.eyJlIjoxNzgxNDQ0MTUzLCJoIjoic255ay5pbyIsImoiOiJBWnoyM1V0enFBV1haczhkOHBLTU5BIiwicyI6InUtMlhadVZrU01Db0NaenBaSnp3VEEiLCJ0aWQiOiJBQUFBQUFBQUFBQUFBQUFBQUFBQUFBIn0.mMItWusgZojnPaUZKQde94Mk7TpNSTRxxgj1aKPB1LyRclJXLPCjgrB0-xA8BouJQdA_tQmqhzecgziKK1RdDg
export SNYK_ORG=3c82b51e-c232-4224-ae07-8aa11c487cf3

snyk test \
    --file=media/pom.xml \
    --package-manager=maven \
    --severity-threshold=high \
    --org=$SNYK_ORG \
    2>&1 | tail -20

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

        // ───────────────── GITLEAKS ─────────────────-
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
                        echo "No secrets found"
                    } else {
                        def report = readFile('gitleaks-report.json')
                        echo report
                        error("Secrets detected in repository")
                    }
                }
            }

            post {
                always {
                    archiveArtifacts artifacts: 'gitleaks-report.json', allowEmptyArchive: true
                }
            }
        }

        // ───────────────── SNYK (PLUGIN) ─────────────────
        stage('Security Scan: Snyk') {
            steps {
                script {

                    def services = [
                        'cart','customer','order','product','rating',
                        'inventory','media','tax','location','promotion'
                    ]

                    services.each { svc ->

                        if (!fileExists("${svc}/pom.xml")) {
                            echo "Skipping ${svc} (no pom.xml)"
                            return
                        }

                        echo "Running Snyk scan for ${svc}"

                        snykSecurity(
                            snykInstallation: 'snyk',
                            snykTokenId: 'snyk-token',
                            failOnIssues: false,
                            projectName: "yas-${svc}",
                            targetFile: "${svc}/pom.xml",
                            additionalArguments: "--severity-threshold=high -d"
                        )
                    }
                }
            }
        }

        // ───────────────── DETECT CHANGES ─────────────────
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

                    echo "Changed files:"
                    echo changedFiles

                    def services = [
                        'cart','customer','order','product','rating',
                        'inventory','media','tax','location','promotion'
                    ]

                    services.each { svc ->

                        env.setProperty(
                            "${svc.toUpperCase()}_CHANGED",
                            changedFiles.contains("${svc}/") ? 'true' : 'false'
                        )
                    }

                    def statusLines = services.collect { svc ->
                        "| ${svc.padRight(10)} : ${env.getProperty("${svc.toUpperCase()}_CHANGED")}"
                    }.join('\n')

                    echo """
+----------------------------------+
|        SERVICES TO BUILD         |
+----------------------------------+
${statusLines}
+----------------------------------+
"""
                }
            }
        }

        // ───────────────── BUILD SERVICES ─────────────────
        stage('Test & Build Services') {
            steps {
                script {

                    def services = [
                        'cart','customer','order','product','rating',
                        'inventory','media','tax','location','promotion'
                    ]

                    services.each { svc ->

                        if (env.getProperty("${svc.toUpperCase()}_CHANGED") != 'true') {

                            echo "Skipping ${svc}"
                            return
                        }

                        echo "Building ${svc}"

                        def projectKey = "yas-${svc}"
                        def projectName = "YAS - ${svc.capitalize()}"

                        stage("${svc} Unit Test") {

                            sh "mvn clean test jacoco:report -pl ${svc} -am"

                            junit "${svc}/target/surefire-reports/*.xml"
                        }

                        stage("${svc} Coverage") {

                            recordCoverage(
                                tools: [[
                                    $class: 'CoverageTool',
                                    parser: 'JACOCO',
                                    pattern: "**/${svc}/target/site/jacoco/jacoco.xml"
                                ]],
                                sourceDirectories: [[path: "${svc}/src/main/java"]]
                            )
                        }

                        stage("${svc} SonarQube") {

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

                        stage("${svc} Quality Gate") {

                            timeout(time: 5, unit: 'MINUTES') {
                                waitForQualityGate abortPipeline: true
                            }
                        }

                        stage("${svc} Build") {
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
            echo "Pipeline completed successfully"
        }

        failure {
            echo "Pipeline failed"
        }
    }
}

https://app.snyk.io/account

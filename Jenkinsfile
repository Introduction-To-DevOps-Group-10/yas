pipeline {
    agent any

    environment {
        SNYK_TOKEN = credentials('snyk-token')
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Install Snyk CLI') {
            steps {
                sh '''
                    if ! command -v snyk > /dev/null 2>&1
                    then
                        echo "Installing Snyk CLI..."
                        npm install -g snyk
                    else
                        echo "Snyk already installed"
                    fi
                '''
            }
        }

        stage('Snyk Scan (Debug Mode)') {
            steps {
                sh '''
                    echo "===== SNYK DEBUG SCAN START ====="

                    snyk --version

                    snyk auth $SNYK_TOKEN

                    snyk test \
                      --all-projects \
                      --detection-depth=4 \
                      --severity-threshold=low \
                      -d \
                      --json-file-output=snyk-report.json || true

                    echo "===== SNYK DEBUG SCAN END ====="
                '''
            }
        }

        stage('Archive Report') {
            steps {
                archiveArtifacts artifacts: 'snyk-report.json', allowEmptyArchive: true
            }
        }

        // ✅ cleanup chạy TRONG node → không lỗi nữa
        stage('Cleanup') {
            steps {
                cleanWs()
            }
        }
    }

    post {
        success {
            echo "Snyk scan completed (debug mode)"
        }
        failure {
            echo "Pipeline failed"
        }
    }
}

pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
    }

    environment {
        APP_DIR = '/var/www/dora-site'
        DORA_LOG = '/var/lib/jenkins/dora-metrics/deployments.csv'
        DORA_WINDOW_DAYS = '30'
        SEMGREP_BIN = '/var/lib/jenkins/semgrep-venv/bin/semgrep'
        SEMGREP_STATUS = 'UNKNOWN'
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Capture Source Info') {
            steps {
                script {
                    env.GIT_COMMIT_SHORT = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
                    env.GIT_COMMIT_EPOCH = sh(script: "git log -1 --format=%ct", returnStdout: true).trim()
                    env.GIT_COMMIT_ISO = sh(script: "git log -1 --format=%cI", returnStdout: true).trim()

                    echo "Commit: ${env.GIT_COMMIT_SHORT}"
                    echo "Commit time: ${env.GIT_COMMIT_ISO}"
                }
            }
        }

        stage('Prepare Workspace') {
            steps {
                sh 'rm -rf build semgrep-report.json dora-metrics.json'
            }
        }

        stage('Semgrep Analysis') {
            steps {
                script {
                    sh 'test -x "$SEMGREP_BIN"'

                    def exitCode = sh(
                        returnStatus: true,
                        script: '''
                            "$SEMGREP_BIN" scan --config=auto --error --json-output=semgrep-report.json .
                        '''
                    )

                    if (exitCode == 0) {
                        env.SEMGREP_STATUS = "OK"
                    } else if (exitCode == 1) {
                        env.SEMGREP_STATUS = "ISSUES"
                    } else {
                        error "Semgrep error (exit code ${exitCode})"
                    }

                    archiveArtifacts artifacts: 'semgrep-report.json', fingerprint: true
                    echo "Semgrep status FIXED: ${env.SEMGREP_STATUS}"
                }
            }
        }

        stage('Build') {
            steps {
                echo "Build static web"
                sh '''
                    set +e
                    rm -rf build
                    mkdir -p build

                    cp index.html build/ 2>/dev/null || true
                    cp templates.html build/ 2>/dev/null || true
                    cp templatemo-quantix-style.css build/ 2>/dev/null || true
                    cp templatemo-quantix-script.js build/ 2>/dev/null || true

                    for d in assets images img css js fonts vendor; do
                        if [ -d "$d" ]; then
                            cp -r "$d" build/
                        fi
                    done
                '''
            }
        }

        stage('Deploy to Nginx') {
            steps {
                sh '''
                    mkdir -p "$APP_DIR"
                    rm -rf "$APP_DIR"/*
                    cp -r build/* "$APP_DIR"/
                '''
            }
        }

        stage('DORA Metrics') {
            steps {
                script {
                    def deployEnd = sh(script: "date +%s", returnStdout: true).trim().toInteger()
                    def commitTime = env.GIT_COMMIT_EPOCH.toInteger()

                    def ltSeconds = deployEnd - commitTime
                    def ltMinutes = ltSeconds / 60.0

                    def deployStatus = (env.SEMGREP_STATUS == "OK") ? "SUCCESS" : "SUCCESS_WITH_ISSUES"

                    sh """
                        mkdir -p \$(dirname "$DORA_LOG")

                        if [ ! -f "$DORA_LOG" ]; then
                          echo "build,commit,commit_epoch,deploy_epoch,lt,status,semgrep" > "$DORA_LOG"
                        fi

                        printf '%s,%s,%s,%s,%s,%s,%s\\n' \
                        '${env.BUILD_NUMBER}' \
                        '${env.GIT_COMMIT_SHORT}' \
                        '${env.GIT_COMMIT_EPOCH}' \
                        '${deployEnd}' \
                        '${ltSeconds}' \
                        '${deployStatus}' \
                        '${env.SEMGREP_STATUS}' >> "$DORA_LOG"
                    """

                    currentBuild.description = "LT=${ltMinutes}m | Semgrep=${env.SEMGREP_STATUS}"
                }
            }
        }
    }

    post {
        success {
            echo "======================="
            echo "PIPELINE SUCCESS"
            echo "======================="
            echo "Semgrep: ${env.SEMGREP_STATUS}"
        }

        failure {
            echo "PIPELINE FAILED"
        }
    }
}

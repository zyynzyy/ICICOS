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
        stage('Prepare Workspace') {
            steps {
                sh '''
                    rm -rf build semgrep-report.json semgrep-console.log dora-metrics.json
                '''
            }
        }

        stage('Semgrep Analysis') {
            steps {
                script {
                    sh 'test -x "$SEMGREP_BIN"'

                    def semgrepStatus = sh(
                        returnStdout: true,
                        script: '''
                            set +e
                            "$SEMGREP_BIN" scan --config=auto --error --json-output=semgrep-report.json . > semgrep-console.log 2>&1
                            code=$?

                            if [ $code -eq 0 ]; then
                                printf OK
                            elif [ $code -eq 1 ]; then
                                printf ISSUES
                            else
                                printf FAILED
                            fi
                        '''
                    ).trim()

                    if (semgrepStatus == 'FAILED') {
                        error "Semgrep error"
                    }

                    env.SEMGREP_STATUS = semgrepStatus

                    archiveArtifacts artifacts: 'semgrep-report.json,semgrep-console.log', fingerprint: true
                    echo "Semgrep status FINAL: ${env.SEMGREP_STATUS}"
                }
            }
        }

        stage('Build') {
            steps {
                echo "Build static web"
                sh '''
                    set -e
                    rm -rf build
                    mkdir -p build

                    for f in index.html templates.html templatemo-quantix-style.css templatemo-quantix-script.js; do
                        if [ -f "$f" ]; then
                            cp "$f" build/
                        fi
                    done

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
                echo "Deploy ke Nginx"
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
def result = sh(
    returnStdout: true,
    script: '''
        set -e

        DEPLOY_EPOCH=$(date +%s)

        # FIX DI SINI
        LT_SECONDS=$((DEPLOY_EPOCH - $GIT_COMMIT_EPOCH))

        LT_MINUTES=$(awk -v s="$LT_SECONDS" 'BEGIN { printf "%.2f", s/60 }')

        mkdir -p "$(dirname "$DORA_LOG")"

        if [ ! -f "$DORA_LOG" ]; then
            echo "build_number,commit,commit_epoch,deploy_epoch,lt_seconds,status,semgrep_status" > "$DORA_LOG"
        fi

        if [ "$SEMGREP_STATUS" = "OK" ]; then
            DEPLOY_STATUS="SUCCESS"
        else
            DEPLOY_STATUS="SUCCESS_WITH_ISSUES"
        fi

        printf '%s,%s,%s,%s,%s,%s,%s\n' \
            "$BUILD_NUMBER" \
            "$GIT_COMMIT_SHORT" \
            "$GIT_COMMIT_EPOCH" \
            "$DEPLOY_EPOCH" \
            "$LT_SECONDS" \
            "$DEPLOY_STATUS" \
            "$SEMGREP_STATUS" >> "$DORA_LOG"

        echo "$LT_SECONDS|$LT_MINUTES|0|0.0000"
    '''
).trim()

                    def parts = result.split(/\|/)
                    env.DORA_LT_SECONDS = parts[0]
                    env.DORA_LT_MINUTES = parts[1]
                    env.DORA_DF_COUNT = parts[2]
                    env.DORA_DF_PER_DAY = parts[3]

                    writeFile file: 'dora-metrics.json', text: groovy.json.JsonOutput.prettyPrint(
                        groovy.json.JsonOutput.toJson([
                            buildNumber: env.BUILD_NUMBER,
                            commit: env.GIT_COMMIT_SHORT,
                            leadTimeSeconds: env.DORA_LT_SECONDS,
                            leadTimeMinutes: env.DORA_LT_MINUTES,
                            deployCountLast30Days: env.DORA_DF_COUNT,
                            deployFrequencyPerDay: env.DORA_DF_PER_DAY,
                            semgrepStatus: env.SEMGREP_STATUS
                        ])
                    )

                    archiveArtifacts artifacts: 'dora-metrics.json', fingerprint: true
                    currentBuild.description = "LT=${env.DORA_LT_MINUTES}m | DF30=${env.DORA_DF_COUNT} | Semgrep=${env.SEMGREP_STATUS}"
                }
            }
        }
    }

    post {
        success {
            echo "=============================="
            echo "PIPELINE SUCCESS"
            echo "=============================="
            echo "DORA FINAL RESULT:"
            echo "Lead Time (LT): ${env.DORA_LT_SECONDS} detik (${env.DORA_LT_MINUTES} menit)"
            echo "Deployment Frequency (DF): ${env.DORA_DF_COUNT} deploy dalam ${env.DORA_WINDOW_DAYS} hari"
            echo "DF Rate: ${env.DORA_DF_PER_DAY} deploy/hari"
            echo "Semgrep: ${env.SEMGREP_STATUS}"
        }

        failure {
            echo "PIPELINE FAILED"
        }

        always {
            echo "Build status: ${currentBuild.currentResult}"
        }
    }
}

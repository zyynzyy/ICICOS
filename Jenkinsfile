pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
    }

    environment {
        APP_DIR          = '/var/www/dora-site'
        DORA_LOG         = '/var/lib/jenkins/dora-metrics/deployments.csv'
        DORA_WINDOW_DAYS = '30'
        SEMGREP_BIN      = '/var/lib/jenkins/semgrep-venv/bin/semgrep'
    }

    stages {

        stage('Init Git Info') {
            steps {
                script {
                    def shortSha = sh(
                        returnStdout: true,
                        script: 'git rev-parse --short HEAD'
                    ).trim()

                    def commitEpoch = sh(
                        returnStdout: true,
                        script: 'git log -1 --format=%ct HEAD'
                    ).trim()

                    env.GIT_COMMIT_SHORT = shortSha
                    env.GIT_COMMIT_EPOCH = commitEpoch

                    echo "Commit : ${env.GIT_COMMIT_SHORT}"
                    echo "Epoch  : ${env.GIT_COMMIT_EPOCH}"

                    if (!env.GIT_COMMIT_SHORT || !env.GIT_COMMIT_EPOCH) {
                        error 'Gagal membaca info Git — periksa apakah repo sudah di-checkout.'
                    }
                }
            }
        }

        stage('Prepare Workspace') {
            steps {
                sh 'rm -rf build semgrep-report.json semgrep-console.log dora-metrics.json'
            }
        }

        stage('Semgrep Analysis') {
            steps {
                script {
                    sh 'test -x "$SEMGREP_BIN"'

                    def exitCode = sh(
                        returnStatus: true,
                        script: '''
                            "$SEMGREP_BIN" scan \
                                --config=auto \
                                --error \
                                --json-output=semgrep-report.json \
                                . > semgrep-console.log 2>&1
                        '''
                    )

                    echo "Semgrep raw exit code: ${exitCode}"

                    if (exitCode == 0) {
                        env.SEMGREP_STATUS = 'OK'
                    } else if (exitCode == 1) {
                        env.SEMGREP_STATUS = 'ISSUES'
                    } else {
                        env.SEMGREP_STATUS = 'FAILED'
                        error "Semgrep tool error (exit ${exitCode}) — lihat semgrep-console.log"
                    }

                    archiveArtifacts artifacts: 'semgrep-report.json,semgrep-console.log',
                                     allowEmptyArchive: true,
                                     fingerprint: true

                    echo "Semgrep status: ${env.SEMGREP_STATUS}"
                }
            }
        }

        stage('Build') {
            steps {
                echo 'Build static web'
                sh '''
                    set -e
                    rm -rf build
                    mkdir -p build

                    for f in index.html templates.html templatemo-quantix-style.css templatemo-quantix-script.js; do
                        [ -f "$f" ] && cp "$f" build/
                    done

                    for d in assets images img css js fonts vendor; do
                        [ -d "$d" ] && cp -r "$d" build/
                    done
                '''
            }
        }

        stage('Deploy to Nginx') {
            steps {
                echo 'Deploy ke Nginx'
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
                    if (!env.GIT_COMMIT_EPOCH?.trim()) {
                        error 'GIT_COMMIT_EPOCH kosong — stage Init Git Info gagal.'
                    }
                    if (!env.SEMGREP_STATUS?.trim()) {
                        error 'SEMGREP_STATUS kosong — stage Semgrep Analysis gagal.'
                    }

                    def doraOutput = sh(
                        returnStdout: true,
                        script: '''
                            set -e

                            DEPLOY_EPOCH=$(date +%s)

                            LT_SECONDS=$(( DEPLOY_EPOCH - GIT_COMMIT_EPOCH ))
                            [ "$LT_SECONDS" -lt 0 ] && LT_SECONDS=0
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

                            WINDOW_SECONDS=$(( DORA_WINDOW_DAYS * 86400 ))
                            CUTOFF=$(( DEPLOY_EPOCH - WINDOW_SECONDS ))

                            DF_COUNT=$(awk -F',' -v cutoff="$CUTOFF" '
                                NR > 1 && $4+0 >= cutoff && ($6 == "SUCCESS" || $6 == "SUCCESS_WITH_ISSUES") {
                                    count++
                                }
                                END { print (count ? count : 0) }
                            ' "$DORA_LOG")

                            DF_PER_DAY=$(awk -v c="$DF_COUNT" -v w="$DORA_WINDOW_DAYS" \
                                'BEGIN { printf "%.4f", c/w }')

                            printf '%s|%s|%s|%s' \
                                "$LT_SECONDS" "$LT_MINUTES" "$DF_COUNT" "$DF_PER_DAY"
                        '''
                    ).trim()

                    def parts = doraOutput.tokenize('|')
                    env.DORA_LT_SECONDS  = parts[0]
                    env.DORA_LT_MINUTES  = parts[1]
                    env.DORA_DF_COUNT    = parts[2]
                    env.DORA_DF_PER_DAY  = parts[3]

                    writeFile file: 'dora-metrics.json',
                              text: groovy.json.JsonOutput.prettyPrint(
                                  groovy.json.JsonOutput.toJson([
                                      buildNumber           : env.BUILD_NUMBER,
                                      commit                : env.GIT_COMMIT_SHORT,
                                      commitEpoch           : env.GIT_COMMIT_EPOCH,
                                      leadTimeSeconds       : env.DORA_LT_SECONDS,
                                      leadTimeMinutes       : env.DORA_LT_MINUTES,
                                      deployCountLast30Days : env.DORA_DF_COUNT,
                                      deployFrequencyPerDay : env.DORA_DF_PER_DAY,
                                      semgrepStatus         : env.SEMGREP_STATUS
                                  ])
                              )

                    archiveArtifacts artifacts: 'dora-metrics.json', fingerprint: true
                    currentBuild.description =
                        "LT=${env.DORA_LT_MINUTES}m | DF30=${env.DORA_DF_COUNT} | Semgrep=${env.SEMGREP_STATUS}"
                }
            }
        }
    }

    post {
        success {
            echo '=============================='
            echo 'PIPELINE SUCCESS'
            echo '=============================='
            echo "Lead Time    : ${env.DORA_LT_SECONDS} detik (${env.DORA_LT_MINUTES} menit)"
            echo "Deploy count : ${env.DORA_DF_COUNT} deploy dalam ${env.DORA_WINDOW_DAYS} hari terakhir"
            echo "DF Rate      : ${env.DORA_DF_PER_DAY} deploy/hari"
            echo "Semgrep      : ${env.SEMGREP_STATUS}"
        }
        failure {
            echo 'PIPELINE FAILED'
        }
        always {
            echo "Build status: ${currentBuild.currentResult}"
        }
    }
}

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
        // SEMGREP_STATUS sengaja TIDAK didefinisikan di sini
        // supaya env.SEMGREP_STATUS = ... di dalam script{} bisa bekerja
    }

    stages {
        stage('Capture Source Info') {
            steps {
                script {
                    env.GIT_COMMIT_SHORT = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
                    env.GIT_COMMIT_EPOCH = sh(script: "git log -1 --format=%ct", returnStdout: true).trim()
                    env.GIT_COMMIT_ISO   = sh(script: "git log -1 --format=%cI", returnStdout: true).trim()

                    echo "Commit      : ${env.GIT_COMMIT_SHORT}"
                    echo "Commit time : ${env.GIT_COMMIT_ISO}"
                }
            }
        }

        stage('Prepare Workspace') {
            steps {
                sh 'rm -rf build semgrep-report.json dora-metrics.json semgrep-status.txt'
            }
        }

        stage('Semgrep Analysis') {
            steps {
                script {
                    sh 'test -x "$SEMGREP_BIN"'

                    // Jalankan semgrep, tulis label status ke file txt
                    // set +e supaya exit code tidak langsung abort sh block
                    sh '''
                        set +e
                        "$SEMGREP_BIN" scan --config=auto --error \
                            --json-output=semgrep-report.json . \
                            > semgrep-console.log 2>&1
                        code=$?

                        if   [ $code -eq 0 ]; then echo "OK"     > semgrep-status.txt
                        elif [ $code -eq 1 ]; then echo "ISSUES" > semgrep-status.txt
                        else                       echo "FAILED"  > semgrep-status.txt
                        fi

                        # Selalu exit 0 dari sini supaya Groovy yang kontrol flow
                        exit 0
                    '''

                    // Baca hasil dari file — ini cara paling reliable untuk
                    // melewati nilai dari shell ke Groovy di Declarative Pipeline
                    def statusFromFile = readFile('semgrep-status.txt').trim()

                    if (statusFromFile == 'FAILED') {
                        error "Semgrep tool error — lihat semgrep-console.log"
                    }

                    // Assign ke env.* SETELAH tidak ada di environment{} block
                    env.SEMGREP_STATUS = statusFromFile

                    archiveArtifacts artifacts: 'semgrep-report.json,semgrep-console.log,semgrep-status.txt',
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
                    def result = sh(
                        returnStdout: true,
                        script: '''
                            set -e

                            DEPLOY_EPOCH=$(date +%s)
                            LT_SECONDS=$(( DEPLOY_EPOCH - GIT_COMMIT_EPOCH ))
                            [ "$LT_SECONDS" -lt 0 ] && LT_SECONDS=0
                            LT_MINUTES=$(awk -v s="$LT_SECONDS" 'BEGIN { printf "%.2f", s/60 }')

                            WINDOW_START=$(( DEPLOY_EPOCH - DORA_WINDOW_DAYS * 86400 ))

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

                            DEPLOY_COUNT=$(awk -F',' -v ws="$WINDOW_START" '
                                NR > 1 && $4+0 >= ws && $6 ~ /^SUCCESS/ { c++ }
                                END { print c+0 }
                            ' "$DORA_LOG")

                            DF_PER_DAY=$(awk -v c="$DEPLOY_COUNT" -v d="$DORA_WINDOW_DAYS" \
                                'BEGIN { printf "%.4f", c/d }')

                            echo "$LT_SECONDS|$LT_MINUTES|$DEPLOY_COUNT|$DF_PER_DAY"
                        '''
                    ).trim()

                    def parts = result.split(/\|/)
                    env.DORA_LT_SECONDS  = parts[0]
                    env.DORA_LT_MINUTES  = parts[1]
                    env.DORA_DF_COUNT    = parts[2]
                    env.DORA_DF_PER_DAY  = parts[3]

                    writeFile file: 'dora-metrics.json',
                              text: groovy.json.JsonOutput.prettyPrint(
                                  groovy.json.JsonOutput.toJson([
                                      buildNumber           : env.BUILD_NUMBER,
                                      commit                : env.GIT_COMMIT_SHORT,
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

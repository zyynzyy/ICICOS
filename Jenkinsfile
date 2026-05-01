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

        // Akan di-populate di stage 'Init Git Info'
        GIT_COMMIT_SHORT = ''
        GIT_COMMIT_EPOCH = ''

        // Diisi oleh stage Semgrep — nilai awal sengaja dikosongkan,
        // bukan 'UNKNOWN', supaya mudah deteksi kalau stage-nya skip
        SEMGREP_STATUS   = ''
    }

    stages {

        // ----------------------------------------------------------------
        // Stage 0 – ambil info Git sekali, simpan di env supaya semua
        //           stage berikutnya bisa pakai tanpa hitung ulang
        // ----------------------------------------------------------------
        stage('Init Git Info') {
            steps {
                script {
                    // GIT_COMMIT sudah disediakan Jenkins secara otomatis
                    env.GIT_COMMIT_SHORT = sh(
                        returnStdout: true,
                        script: 'git rev-parse --short HEAD'
                    ).trim()

                    // Unix-epoch dari commit terakhir
                    env.GIT_COMMIT_EPOCH = sh(
                        returnStdout: true,
                        script: 'git log -1 --format=%ct HEAD'
                    ).trim()

                    echo "Commit : ${env.GIT_COMMIT_SHORT}"
                    echo "Epoch  : ${env.GIT_COMMIT_EPOCH}"
                }
            }
        }

        stage('Prepare Workspace') {
            steps {
                sh 'rm -rf build semgrep-report.json semgrep-console.log dora-metrics.json'
            }
        }

        // ----------------------------------------------------------------
        // Stage Semgrep
        // Kode keluar  0  → OK   (tidak ada temuan)
        // Kode keluar  1  → ISSUES (ada temuan, pipeline tetap lanjut)
        // Kode keluar selain itu → FAILED (error tool) → abort
        // ----------------------------------------------------------------
        stage('Semgrep Analysis') {
            steps {
                script {
                    sh 'test -x "$SEMGREP_BIN"'

                    // Jalankan semgrep, tangkap exit-code lewat file supaya
                    // tidak bergantung pada pipe yang bisa menelan kode
                    def semgrepResult = sh(
                        returnStatus: true,          // <-- returnStatus, bukan returnStdout
                        script: '''
                            "$SEMGREP_BIN" scan \
                                --config=auto \
                                --error \
                                --json-output=semgrep-report.json \
                                . > semgrep-console.log 2>&1
                        '''
                    )

                    // Petakan exit-code → label
                    if (semgrepResult == 0) {
                        env.SEMGREP_STATUS = 'OK'
                    } else if (semgrepResult == 1) {
                        env.SEMGREP_STATUS = 'ISSUES'
                    } else {
                        // Tool gagal (misal: config error, crash) → hentikan build
                        error "Semgrep exited with code ${semgrepResult} — lihat semgrep-console.log"
                    }

                    archiveArtifacts artifacts: 'semgrep-report.json,semgrep-console.log',
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

        // ----------------------------------------------------------------
        // DORA Metrics
        // Lead Time = waktu antara commit masuk repo s.d. deploy selesai
        // Deployment Frequency = jumlah deploy sukses dalam 30 hari terakhir
        // ----------------------------------------------------------------
        stage('DORA Metrics') {
            steps {
                script {
                    // Pastikan GIT_COMMIT_EPOCH sudah terisi
                    if (!env.GIT_COMMIT_EPOCH?.trim()) {
                        error 'GIT_COMMIT_EPOCH kosong — pastikan stage Init Git Info berhasil'
                    }

                    def doraResult = sh(
                        returnStdout: true,
                        script: '''
                            set -e

                            DEPLOY_EPOCH=$(date +%s)

                            # Lead Time dalam detik (commit → deploy)
                            LT_SECONDS=$(( DEPLOY_EPOCH - GIT_COMMIT_EPOCH ))

                            # Hindari nilai negatif (clock skew / commit di masa depan)
                            [ "$LT_SECONDS" -lt 0 ] && LT_SECONDS=0

                            LT_MINUTES=$(awk -v s="$LT_SECONDS" 'BEGIN { printf "%.2f", s/60 }')

                            # ---------- tulis ke CSV log ----------
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

                            # ---------- hitung Deployment Frequency ----------
                            # Hitung baris deploy SUCCESS atau SUCCESS_WITH_ISSUES
                            # dalam window 30 hari terakhir
                            WINDOW_SECONDS=$(( DORA_WINDOW_DAYS * 86400 ))
                            CUTOFF=$(( DEPLOY_EPOCH - WINDOW_SECONDS ))

                            DF_COUNT=$(awk -F',' -v cutoff="$CUTOFF" '
                                NR > 1 && $4 >= cutoff && ($6 == "SUCCESS" || $6 == "SUCCESS_WITH_ISSUES") {
                                    count++
                                }
                                END { print (count ? count : 0) }
                            ' "$DORA_LOG")

                            DF_PER_DAY=$(awk -v c="$DF_COUNT" -v w="$DORA_WINDOW_DAYS" \
                                'BEGIN { printf "%.4f", c/w }')

                            # Kembalikan semua nilai dalam satu baris, dipisah |
                            printf '%s|%s|%s|%s' \
                                "$LT_SECONDS" "$LT_MINUTES" "$DF_COUNT" "$DF_PER_DAY"
                        '''
                    ).trim()

                    def parts = doraResult.tokenize('|')
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
            echo "DORA FINAL RESULT:"
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

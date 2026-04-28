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

        SONAR_PROJECT_KEY = 'zyynzyy_Jenkins-CI-CD-SonarQube'
        SONAR_ORG = 'zyynzyy'
        SONAR_TOKEN = credentials('sonar-token')

        SONAR_QG_STATUS = 'UNKNOWN'
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

        stage('Build') {
            steps {
                echo "Build static web"
                sh '''
                    rm -rf build
                    mkdir -p build
                    cp -r index.html assets build/
                '''
            }
        }

        stage('SonarCloud Analysis') {
            steps {
                script {
                    def scannerHome = tool 'sonar-scanner'

                    withSonarQubeEnv('sonarcloud') {
                        sh '''
                            '"${scannerHome}"'/bin/sonar-scanner \
                              -Dsonar.projectKey=$SONAR_PROJECT_KEY \
                              -Dsonar.organization=$SONAR_ORG \
                              -Dsonar.sources=. \
                              -Dsonar.host.url=https://sonarcloud.io \
                              -Dsonar.token=$SONAR_TOKEN
                        '''
                    }
                }
            }
        }

        stage('Quality Gate (Non-blocking)') {
            steps {
                script {
                    echo "Menunggu hasil analisis SonarCloud..."

                    def status = "UNKNOWN"

                    for (int i = 0; i < 3; i++) {
                        sleep 15

                        status = sh(
                            script: '''
                                curl -s -u $SONAR_TOKEN: \
                                "https://sonarcloud.io/api/qualitygates/project_status?projectKey=$SONAR_PROJECT_KEY" \
                                | grep -o '"status":"[^"]*"' | head -1 | cut -d':' -f2 | tr -d '"'
                            ''',
                            returnStdout: true
                        ).trim()

                        if (status == "OK" || status == "ERROR") {
                            break
                        }
                    }

                    env.SONAR_QG_STATUS = status

                    echo "Quality Gate Final Status: ${status}"

                    if (status != "OK") {
                        echo "⚠️ Quality Gate FAILED (non-blocking)"
                    } else {
                        echo "✅ Quality Gate PASSED"
                    }
                }
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

                    sh '''
                        DEPLOY_EPOCH=$(date +%s)
                        COMMIT_EPOCH=$GIT_COMMIT_EPOCH

                        LT_SECONDS=$((DEPLOY_EPOCH - COMMIT_EPOCH))

                        if [ "$SONAR_QG_STATUS" = "OK" ]; then
                            STATUS="SUCCESS"
                        else
                            STATUS="SUCCESS_WITH_ISSUES"
                        fi

                        mkdir -p $(dirname "$DORA_LOG")

                        if [ ! -f "$DORA_LOG" ]; then
                            echo "build_number,commit,commit_epoch,deploy_epoch,lt_seconds,status,qg_status" > "$DORA_LOG"
                        fi

                        echo "$BUILD_NUMBER,$GIT_COMMIT_SHORT,$COMMIT_EPOCH,$DEPLOY_EPOCH,$LT_SECONDS,$STATUS,$SONAR_QG_STATUS" >> "$DORA_LOG"

                        WINDOW_START=$(date -d "$DORA_WINDOW_DAYS days ago" +%s)

                        DEPLOY_COUNT=$(awk -F',' -v ws=$WINDOW_START '
                            NR > 1 && $4 >= ws && $6 ~ /^SUCCESS/ { c++ }
                            END { print c+0 }
                        ' "$DORA_LOG")

                        DF_PER_DAY=$(awk -v c=$DEPLOY_COUNT -v d=$DORA_WINDOW_DAYS 'BEGIN { printf "%.4f", c/d }')
                        LT_MINUTES=$(awk -v s=$LT_SECONDS 'BEGIN { printf "%.2f", s/60 }')

                        echo "LT_SECONDS=$LT_SECONDS" > dora.env
                        echo "LT_MINUTES=$LT_MINUTES" >> dora.env
                        echo "DEPLOY_COUNT=$DEPLOY_COUNT" >> dora.env
                        echo "DF_PER_DAY=$DF_PER_DAY" >> dora.env
                    '''

                    def props = readProperties file: 'dora.env'

                    env.DORA_LT_SECONDS = props.LT_SECONDS
                    env.DORA_LT_MINUTES = props.LT_MINUTES
                    env.DORA_DF_COUNT = props.DEPLOY_COUNT
                    env.DORA_DF_PER_DAY = props.DF_PER_DAY

                    writeFile file: 'dora-metrics.json', text: groovy.json.JsonOutput.prettyPrint(
                        groovy.json.JsonOutput.toJson([
                            buildNumber: env.BUILD_NUMBER,
                            commit: env.GIT_COMMIT_SHORT,
                            leadTimeSeconds: env.DORA_LT_SECONDS,
                            leadTimeMinutes: env.DORA_LT_MINUTES,
                            deployCountLast30Days: env.DORA_DF_COUNT,
                            deployFrequencyPerDay: env.DORA_DF_PER_DAY,
                            qualityGate: env.SONAR_QG_STATUS
                        ])
                    )

                    archiveArtifacts artifacts: 'dora-metrics.json', fingerprint: true

                    currentBuild.description = "LT=${env.DORA_LT_MINUTES}m | DF30=${env.DORA_DF_COUNT} | QG=${env.SONAR_QG_STATUS}"
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
            echo "Quality Gate: ${env.SONAR_QG_STATUS}"
        }

        failure {
            echo "PIPELINE FAILED"
        }

        always {
            echo "Build status: ${currentBuild.currentResult}"
        }
    }
}

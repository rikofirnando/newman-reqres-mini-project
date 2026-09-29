pipeline {
    agent { label 'jenkins-agent-01' }

    options {
        skipDefaultCheckout(true)
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '14'))
    }

    // Aktifkan setelah Build Now berhasil. Jam mengikuti zona waktu Jenkins controller.
    // triggers { cron('H 8 * * *') }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Check Newman') {
            steps {
                sh '''#!/usr/bin/env bash
set -Eeuo pipefail
node --version
npm --version
test -f postman/reqres.collection.json
'''
            }
        }

        stage('Install Newman') {
            steps {
                sh '''#!/usr/bin/env bash
set -Eeuo pipefail
npm install --prefix .newman-tools --no-save --no-package-lock --no-audit --no-fund newman@6.2.2
.newman-tools/node_modules/.bin/newman --version
'''
            }
        }

        stage('Run API Tests') {
            steps {
                withCredentials([string(credentialsId: 'reqres-api-key', variable: 'REQRES_API_KEY')]) {
                    sh '''#!/usr/bin/env bash
set -Eeuo pipefail
set +x
umask 077
.newman-tools/node_modules/.bin/newman run postman/reqres.collection.json \
mkdir -p reports
newman run postman/reqres.collection.json \
  --env-var "api_key=$REQRES_API_KEY" \
  --reporters cli,junit \
  --reporter-junit-export reports/newman.xml \
  --timeout-request 30000
'''
                }
            }
        }
    }

    post {
        always {
            script {
                if (fileExists('reports/newman.xml')) {
                    junit testResults: 'reports/newman.xml'
                }
            }
        }
    }
}

pipeline {
    agent none

    options {
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '14'))
    }

    stages {
        stage('Newman API Test') {
            agent { label 'jenkins-agent-01' }

            options {
                skipDefaultCheckout(true)
            }

            steps {
                checkout scm

                echo '========================================'
                echo 'MENJALANKAN API TEST MENGGUNAKAN NEWMAN'
                echo '========================================'
                echo "Job Name     : ${env.JOB_NAME}"
                echo "Build Number : ${env.BUILD_NUMBER}"
                echo "Node         : ${env.NODE_NAME}"
                echo "Workspace    : ${env.WORKSPACE}"

                sh '''#!/usr/bin/env bash
set -Eeuo pipefail

echo 'Node.js version:'
node --version
echo 'NPM version:'
npm --version

test -f postman/reqres.collection.json

npm install --prefix .newman-tools \
  --no-save --no-package-lock --no-audit --no-fund \
  newman@6.2.2

.newman-tools/node_modules/.bin/newman --version
mkdir -p newman
'''

                withCredentials([
                    string(credentialsId: 'reqres-api-key', variable: 'REQRES_API_KEY')
                ]) {
                    sh '''#!/usr/bin/env bash
set -Eeuo pipefail
set +x
umask 077

.newman-tools/node_modules/.bin/newman run postman/reqres.collection.json \
  --env-var "api_key=$REQRES_API_KEY" \
  --reporters cli,junit \
  --reporter-junit-export newman/results.xml \
  --timeout-request 30000
'''
                }
            }

            post {
                always {
                    script {
                        if (fileExists('newman/results.xml')) {
                            junit testResults: 'newman/results.xml'
                        }
                    }
                }

                success {
                    echo 'Newman API Test berhasil'
                }

                failure {
                    echo 'Newman API Test gagal. Periksa Console Output.'
                }
            }
        }
    }

    post {
        always {
            echo "Status akhir Pipeline: ${currentBuild.currentResult}"
        }
    }
}

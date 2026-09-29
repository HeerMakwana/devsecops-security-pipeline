pipeline {
    agent any

    options {
        timestamps()
    }

    stages {
        stage('Checkout') {
            steps {
                echo 'Retrieving source code from GitHub'
                checkout scm
            }
        }

        stage('Build and Unit Tests') {
            steps {
                sh '''
                    set -e
                    tar --exclude=.git -cf - . |
                        docker build -t devsecops-app:latest -
                '''
            }
        }

        stage('Deploy Application') {
            steps {
                sh '''
                    set -e

                    docker network inspect devsecops-net \
                      >/dev/null 2>&1 ||
                      docker network create devsecops-net

                    docker rm -f devsecops-app || true

                    docker run -d \
                      --name devsecops-app \
                      --network devsecops-net \
                      -p 127.0.0.1:5000:5000 \
                      devsecops-app:latest
                '''
            }
        }

        stage('Health Check') {
            steps {
                sh '''
                    docker run --rm \
                      --network devsecops-net \
                      --entrypoint python \
                      devsecops-app:latest \
                      -c "import urllib.request; print(urllib.request.urlopen('http://devsecops-app:5000/health', timeout=10).read().decode())"
                '''
            }
        }

        stage('OWASP ZAP Scan') {
            steps {
                script {
                    sh 'mkdir -p zap-reports'
                    sh 'docker rm -f zap-scan || true'

                    def scanStatus = sh(
                        script: '''
                            docker run \
                            --name zap-scan \
                            --network devsecops-net \
                            -v zap-scan-data:/zap/wrk:rw \
                            ghcr.io/zaproxy/zaproxy:stable \
                            zap-baseline.py \
                            -t http://devsecops-app:5000 \
                            -r zap-report.html \
                            -J zap-report.json
                        ''',
                        returnStatus: true
                    )

                    echo "ZAP exit code: ${scanStatus}"

                    sh '''
                        docker cp \
                        zap-scan:/zap/wrk/zap-report.html \
                        zap-reports/zap-report.html

                        docker cp \
                        zap-scan:/zap/wrk/zap-report.json \
                        zap-reports/zap-report.json
                    '''

                    if (scanStatus != 0) {
                        echo 'Review ZAP findings and scan logs.'
                        currentBuild.result = 'UNSTABLE'
                    }
                }
            }
        }
    }

    post {
        always {
            archiveArtifacts(
                artifacts: 'zap-reports/*',
                allowEmptyArchive: true
            )

            sh 'docker rm -f zap-scan || true'
        }
    }
}
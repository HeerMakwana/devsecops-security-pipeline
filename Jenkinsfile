pipeline {
    agent any

    options {
        timestamps()
    }

    parameters {
        booleanParam(
            name: 'DEMO_VULNERABLE_DEPENDENCIES',
            defaultValue: false,
            description: 'Use intentionally vulnerable dependencies to demonstrate the security gate'
        )
    }

    stages {

        stage('Checkout') {
            steps {
                echo 'Retrieving source code from GitHub'
                checkout scm
            }
        }

        // ============================================
        // THEORY IA - OWASP DEPENDENCY CHECK
        // ============================================

        stage('OWASP Dependency Check') {
            steps {
                script {
                    sh '''
                        set -e

                        rm -rf dependency-reports
                        mkdir -p dependency-reports

                        docker rm -f dependency-scan || true

                        docker create \
                          --name dependency-scan \
                          -v dependency-check-data:/usr/share/dependency-check/data \
                          owasp/dependency-check:latest \
                          --scan /src \
                          --project "DevSecOps Flask Application" \
                          --format HTML \
                          --format JSON \
                          --out /report \
                          --failOnCVSS 9.0 \
                          --noupdate

                        tar --exclude=.git \
                            --exclude=dependency-reports \
                            --exclude=license-reports \
                            --exclude=zap-reports \
                            -cf - . | \
                          docker cp - dependency-scan:/src

                        docker start -a dependency-scan

                        docker cp \
                          dependency-scan:/report/dependency-check-report.html \
                          dependency-reports/dependency-check-report.html

                        docker cp \
                          dependency-scan:/report/dependency-check-report.json \
                          dependency-reports/dependency-check-report.json

                        docker rm -f dependency-scan
                    '''
                }
            }
        }

        // ============================================
        // THEORY IA - PYTHON VULNERABILITY SCAN
        // ============================================

        stage('Python Dependency Audit') {
            steps {
                script {

                    def requirementsFile = params.DEMO_VULNERABLE_DEPENDENCIES ?
                        'requirements-vulnerable.txt' :
                        'requirements.txt'

                    echo "Scanning dependency file: ${requirementsFile}"

                    sh 'docker rm -f pip-audit-scan || true'

                    sh """
                        docker create \
                        --name pip-audit-scan \
                        python:3.11-slim \
                        sh -c "
                            mkdir -p /reports &&
                            pip install --quiet pip-audit &&
                            pip-audit \
                            -r /requirements.txt \
                            -f json \
                            -o /reports/pip-audit.json
                        "

                        docker cp \
                        ${requirementsFile} \
                        pip-audit-scan:/requirements.txt
                    """

                    def auditStatus = sh(
                        script: 'docker start -a pip-audit-scan',
                        returnStatus: true
                    )

                    echo "pip-audit exit code: ${auditStatus}"

                    sh '''
                        mkdir -p dependency-reports

                        docker cp \
                        pip-audit-scan:/reports/pip-audit.json \
                        dependency-reports/pip-audit.json || true

                        docker rm -f pip-audit-scan || true
                    '''

                    if (auditStatus != 0) {
                        error(
                            "SECURITY GATE FAILED: pip-audit detected known vulnerable Python dependencies."
                        )
                    }

                    echo 'SECURITY GATE PASSED: No known vulnerable Python dependencies detected.'
                }
            }
        }

        // ============================================
        // THEORY IA - LICENSE ANALYSIS
        // ============================================

        stage('License Analysis') {
            steps {
                sh '''
                    set -e

                    rm -rf license-reports
                    mkdir -p license-reports

                    docker rm -f license-scan || true

                    docker create \
                    --name license-scan \
                    python:3.11-slim \
                    sh -c "
                        mkdir -p /reports &&
                        pip install --quiet \
                        -r /requirements.txt \
                        pip-licenses &&
                        pip-licenses \
                        --format=json \
                        --output-file=/reports/licenses.json &&
                        pip-licenses \
                        --format=markdown \
                        --output-file=/reports/licenses.md
                    "

                    docker cp \
                    requirements.txt \
                    license-scan:/requirements.txt

                    docker start -a license-scan

                    docker cp \
                    license-scan:/reports/licenses.json \
                    license-reports/licenses.json

                    docker cp \
                    license-scan:/reports/licenses.md \
                    license-reports/licenses.md

                    docker rm -f license-scan
                '''
            }
        }

        // ============================================
        // EXISTING LAB CA PIPELINE
        // ============================================

        stage('Build and Unit Tests') {
            steps {
                sh '''
                    set -e

                    tar --exclude=.git -cf - . |
                        docker build \
                          -t devsecops-app:latest -
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

        // ============================================
        // LAB CA - OWASP ZAP
        // ============================================

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
                artifacts: 'dependency-reports/*,license-reports/*,zap-reports/*',
                allowEmptyArchive: true
            )

            sh '''
                docker rm -f dependency-scan || true
                docker rm -f pip-audit-scan || true
                docker rm -f license-scan || true
                docker rm -f zap-scan || true
            '''
        }
    }
}
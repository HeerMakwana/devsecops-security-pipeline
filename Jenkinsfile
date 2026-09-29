pipeline {
    agent any

    stages {
        stage('Checkout') {
            steps {
                echo 'GitHub checkout successful!'
            }
        }

        stage('Verify Environment') {
            steps {
                sh 'java -version'
                echo 'Jenkins environment verified!'
            }
        }

        stage('Security Preparation') {
            steps {
                echo 'Preparing OWASP integration!'
            }
        }
    }

    post {
        success {
            echo 'Pipeline completed successfully!'
        }
        failure {
            echo 'Pipeline failed!'
        }
    }
}
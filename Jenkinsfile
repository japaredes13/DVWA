pipeline {
    agent none
    stages {
        stage('Checkout') {
            agent {
                docker { image 'php:8.2-cli' }
            }
            steps {
                sh 'php --version'
            }
        }
        stage('Compilation') {
            agent {
                docker { image 'php:8.2-cli' }
            }
            steps {
                sh 'echo "Compilando..."'
            }
        }
        stage('Build') {
            agent {
                docker { image 'php:8.2-cli' }
            }
            steps {
                sh 'echo "docker build -t my-php-app ."'
            }
        }
        stage('Semgrep Security Scan') {
            steps {
                // Usando contenedor oficial
                sh '''
                docker run --rm -v $PWD:/src returntocorp/semgrep semgrep \
                    --config "p/owasp-top-ten" \
                    --error \
                    --json > semgrep-report.json
                '''
            }
        }

        stage('Publish Report') {
            steps {
                publishHTML([ 
                    allowMissing: false,
                    alwaysLinkToLastBuild: true,
                    keepAll: true,
                    reportDir: '.',
                    reportFiles: 'semgrep-report.json',
                    reportName: 'Semgrep Security Report'
                ])
            }
        }

        stage('Deploy') {
            agent {
                docker { image 'php:8.2-cli' }
            }
            steps {
                sh 'echo "docker run my-php-app ."'
            }
        }
    }
}
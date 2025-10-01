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
            agent any  // Necesario porque agent none no permite steps que requieren nodo
            steps {
                sh '''
                docker run --rm \
                    -v $WORKSPACE:/src \
                    returntocorp/semgrep semgrep \
                    --config "p/owasp-top-ten" \
                    --error \
                    /src \
                    --json > $WORKSPACE/semgrep-report.json
                '''
            }
        }

        stage('Publish Semgrep Report') {
            agent any
            steps {
                archiveArtifacts artifacts: 'semgrep-report.json', fingerprint: true
                sh 'echo "Semgrep report archived at $WORKSPACE/semgrep-report.json"'
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

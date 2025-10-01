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
            agent any
            steps {
                sh '''
                docker run --rm \
                    -v $WORKSPACE:/src \
                    returntocorp/semgrep semgrep \
                    --config "p/owasp-top-ten" \
                    --error \
                    --json /src > semgrep-report.json
                '''
            }
        }

        stage('Publish Report') {
            agent any   // 🔹 También requiere un nodo
            steps {
                // Si solo quieres guardar el JSON como artefacto
                archiveArtifacts artifacts: 'semgrep-report.json', fingerprint: true

                // Si tienes plugin Warnings NG, puedes interpretar findings así:
                // recordIssues tools: [semgrep(pattern: 'semgrep-report.json')]
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

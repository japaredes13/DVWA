pipeline {
    agent any

    options {
        timestamps()
        buildDiscarder(logRotator(numToKeepStr: '5'))
    }

    stages {
        stage('Checkout') {
            steps {
                cleanWs()
                checkout scm
            }
        }

        stage('Build & Test') {
            agent {
                docker {
                    image 'php:8.2-cli'
                    args '-u root'
                }
            }
            steps {
                sh 'php --version'
                echo "✅ Entorno PHP listo"
            }
        }

        stage('Security Scan - Semgrep') {
            steps {
                script {
                    echo "🔍 Ejecutando Semgrep..."
                    def status = sh(
                        returnStatus: true,
                        script: """
                            docker run --rm \
                            -v "\${WORKSPACE}:/src" \
                            -w /src \
                            returntocorp/semgrep:latest semgrep scan \
                            --config p/security-audit \
                            --config p/owasp-top-ten \
                            --config p/php \
                            --metrics=off \
                            --include '**' .
                        """
                    )

                    if (status != 0) {
                        error("❌ Se detectaron vulnerabilidades con Semgrep")
                    } else {
                        echo "✅ Semgrep no encontró vulnerabilidades críticas"
                    }
                }
            }
        }

        stage('Deploy') {
            when { branch 'main' }
            agent {
                docker {
                    image 'php:8.2-cli'
                    args '-u root'
                }
            }
            steps {
                echo "🚀 Desplegando aplicación..."
                // sh 'docker build -t my-app:${BUILD_NUMBER} .'
                // sh 'docker run -d my-app:${BUILD_NUMBER}'
            }
        }
    }

    post {
        success { echo "✅ Pipeline completado con éxito" }
        failure { echo "❌ Pipeline fallido. Revisa la salida del análisis" }
    }
}

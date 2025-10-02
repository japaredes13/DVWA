pipeline {
    agent any

    options {
        timestamps()
        buildDiscarder(logRotator(numToKeepStr: '5'))
        disableConcurrentBuilds()
        timeout(time: 30, unit: 'MINUTES')
    }

    environment {
        PHP_IMAGE = 'php:8.2-cli'
        SEMGREP_IMAGE = 'returntocorp/semgrep:latest'
    }

    stages {
        stage('Checkout') {
            steps {
                cleanWs()
                checkout scm
                echo "📦 Proyecto clonado en workspace: ${env.WORKSPACE}"
            }
        }

        stage('Build & Test') {
            agent {
                docker {
                    image "${env.PHP_IMAGE}"
                    args '-u root'
                    reuseNode true
                }
            }
            steps {
                sh 'php --version'
                echo "✅ Entorno PHP listo"
            }
        }

        stage('Security Scan') {
            steps {
                script {
                    echo "🔍 Ejecutando Semgrep en todo el proyecto..."

                    // Ejecuta Semgrep sobre todo el workspace
                    def exitCode = sh(
                        returnStatus: true,
                        script: """
                        docker run --rm \
                            -v "\${WORKSPACE}:/src" \
                            -w /src \
                            ${env.SEMGREP_IMAGE} \
                            semgrep scan \
                                --config p/security-audit \
                                --config p/owasp-top-ten \
                                --config p/php \
                                --scan-unknown-extensions \
                                --no-git-ignore \
                                --include '**' \
                                --metrics=off \
                                --verbose \
                                .
                        """
                    )

                    if (exitCode != 0) {
                        error("❌ Build fallido: Semgrep encontró vulnerabilidades o errores")
                    }

                    echo "✅ Semgrep completado. Todos los archivos fueron analizados."
                }
            }
        }

        stage('Deploy') {
            when {
                branch 'main'
            }
            agent {
                docker {
                    image "${env.PHP_IMAGE}"
                    args '-u root'
                    reuseNode true
                }
            }
            steps {
                echo "🚀 Desplegando aplicación..."
                // Ejemplo:
                // sh 'docker build -t my-app:${BUILD_NUMBER} .'
                // sh 'docker run -d -p 8080:80 my-app:${BUILD_NUMBER}'
            }
        }
    }

    post {
        always {
            script {
                echo """
╔═══════════════════════════════════════════════════════════╗
║                  RESUMEN DEL PIPELINE                     ║
╠═══════════════════════════════════════════════════════════╣
║  Estado:   ${currentBuild.currentResult}
║  Build:    #${env.BUILD_NUMBER}
║  Duración: ${currentBuild.durationString.replace(' and counting', '')}
╚═══════════════════════════════════════════════════════════╝
"""
            }
        }
        failure {
            echo "❌ Pipeline fallido. Revisa los logs de Semgrep para detalles."
        }
        success {
            echo "✅ Pipeline completado exitosamente"
        }
    }
}

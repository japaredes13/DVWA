pipeline {
    agent any

    environment {
        // Variables centralizadas para referencia en steps/scripts
        PHP_IMAGE     = 'php:8.2-cli'
        SEMGREP_IMAGE = 'returntocorp/semgrep:latest'
        SEMGREP_RULES = 'p/security-audit,p/owasp-top-ten,p/php'
        SEMGREP_REPORT = 'semgrep-report.json'
    }

    options {
        timestamps()
        skipStagesAfterUnstable()
        buildDiscarder(logRotator(numToKeepStr: '10'))
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
                    image 'php:8.2-cli'   // ✅ hardcodeado, no usar ${env.PHP_IMAGE}
                    args '-u root'
                }
            }
            steps {
                sh 'php --version'
                // sh 'composer install --no-dev --optimize-autoloader'
                // sh 'composer test'
            }
        }

        stage('Security Scan - Semgrep') {
            steps {
                script {
                    sh """
                        docker run --rm \
                          -v "\${WORKSPACE}:/src" \
                          -w /src \
                          ${env.SEMGREP_IMAGE} semgrep scan \
                          --config ${env.SEMGREP_RULES} \
                          --json --output ${env.SEMGREP_REPORT} \
                          --metrics=off \
                          --include '**' .
                    """

                    if (!fileExists(env.SEMGREP_REPORT)) {
                        error("❌ No se generó el reporte de Semgrep")
                    }

                    def report = readJSON file: env.SEMGREP_REPORT
                    def findings = report.results ?: []

                    echo "📊 Vulnerabilidades detectadas: ${findings.size()}"
                    findings.take(5).eachWithIndex { f, i ->
                        echo "[${i+1}] ${f.check_id} ${f.path}:${f.start?.line} - ${f.extra?.message}"
                    }

                    if (findings.any { it.extra?.severity == 'ERROR' }) {
                        error("❌ Vulnerabilidades críticas encontradas")
                    } else if (findings) {
                        currentBuild.result = 'UNSTABLE'
                        echo "⚠️ Vulnerabilidades no críticas detectadas"
                    } else {
                        echo "✅ Sin vulnerabilidades encontradas"
                    }
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: env.SEMGREP_REPORT, allowEmptyArchive: true
                }
            }
        }

        stage('Deploy') {
            when { branch 'main' }
            agent {
                docker {
                    image 'php:8.2-cli'   // ✅ igual que arriba
                    args '-u root'
                }
            }
            steps {
                echo "🚀 Desplegando aplicación..."
                // sh 'docker build -t my-app:${BUILD_NUMBER} .'
                // sh 'docker push my-app:${BUILD_NUMBER}'
            }
        }
    }

    post {
        success { echo "✅ Pipeline completado con éxito" }
        unstable { echo "⚠️ Pipeline marcado como UNSTABLE por findings de seguridad" }
        failure { echo "❌ Pipeline fallido. Revisa el análisis de Semgrep" }
    }
}

pipeline {
    agent any

    environment {
        SEMGREP_IMAGE  = 'returntocorp/semgrep:latest'
        SEMGREP_RULES  = 'p/security-audit,p/owasp-top-ten,p/php'
        SEMGREP_REPORT = 'semgrep-report.json'
        PHP_IMAGE      = 'php:8.2-cli'
    }

    stages {
        stage('Checkout') {
            steps {
                cleanWs()
                checkout scm
            }
        }

        stage('Build & Test') {
            agent { docker { image "${env.PHP_IMAGE}" args '-u root' } }
            steps {
                sh 'php --version'
                // sh 'composer install --no-dev --optimize-autoloader'
                // sh 'composer test'
            }
        }

        stage('Security Scan') {
            steps {
                script {
                    def status = sh(
                        returnStatus: true,
                        script: """
                            docker run --rm \
                              -v "\${WORKSPACE}:/src" \
                              -w /src \
                              ${env.SEMGREP_IMAGE} semgrep scan \
                              --config ${env.SEMGREP_RULES} \
                              --json --output ${env.SEMGREP_REPORT} \
                              --metrics=off .
                        """
                    )
                    if (!fileExists(env.SEMGREP_REPORT)) {
                        error("❌ No se generó el reporte de Semgrep")
                    }
                    def report = readJSON file: env.SEMGREP_REPORT
                    def findings = report.results ?: []

                    echo "📊 Vulnerabilidades: ${findings.size()}"
                    findings.take(5).eachWithIndex { f, i ->
                        echo "[${i+1}] ${f.check_id} ${f.path}:${f.start?.line} - ${f.extra?.message}"
                    }
                    if (findings.size() > 5) echo "... y ${findings.size() - 5} más"

                    if (findings.any { it.extra?.severity == 'ERROR' }) {
                        error("❌ Vulnerabilidades críticas encontradas")
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
            agent { docker { image "${env.PHP_IMAGE}" args '-u root' } }
            steps {
                echo "🚀 Desplegando aplicación..."
                // sh 'docker build -t my-app:${BUILD_NUMBER} .'
                // sh 'docker push my-app:${BUILD_NUMBER}'
            }
        }
    }

    post {
        success { echo "✅ Pipeline completado" }
        failure { echo "❌ Pipeline fallido. Revisa Semgrep." }
    }
}

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

        stage('Security Scan') {
            steps {
                script {
                    echo "🔍 Ejecutando Semgrep..."

                    sh """
                        docker run --rm \
                            -v "\${WORKSPACE}:/src" \
                            -w /src \
                            returntocorp/semgrep:latest \
                            semgrep scan \
                            --config p/security-audit \
                            --config p/owasp-top-ten \
                            --config p/php \
                            --json \
                            --output semgrep-report.json \
                            --metrics=off \
                            --include '**' \
                            .
                    """

                    // Validar si existe el reporte
                    if (!fileExists('semgrep-report.json')) {
                        error("❌ Semgrep no generó reporte JSON")
                    }
                    
                    // Leer reporte
                    def report = readJSON file: 'semgrep-report.json'
                    def findings = report.results ?: []
                    def scanned = report.paths?.scanned ?: []
                    
                    echo "📊 Archivos: ${scanned.size()}"
                    echo "🔍 Vulnerabilidades: ${findings.size()}"
                    
                    // Fallar si hay vulnerabilidades
                    if (findings.size() > 0) {
                        echo "⚠️  Se encontraron ${findings.size()} vulnerabilidades"
                        
                        findings.take(5).each { issue ->
                            echo "  • ${issue.check_id} en ${issue.path}:${issue.start?.line}"
                        }
                        
                        error("❌ Build fallido por vulnerabilidades")
                    }
                    
                    echo "✅ Sin vulnerabilidades"
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

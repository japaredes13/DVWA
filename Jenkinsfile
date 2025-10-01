pipeline {
    agent none
    
    environment {
        SEMGREP_IMAGE = 'returntocorp/semgrep:latest'
        SEMGREP_RULES = 'p/security-audit'
    }
    
    stages {
        stage('Checkout') {
            agent any
            steps {
                checkout scm
                sh 'ls -la'
            }
        }

        stage('Build') {
            agent {
                docker { 
                    image 'php:8.2-cli'
                    args '-u root'
                }
            }
            steps {
                sh 'php --version'
                sh 'echo "Compilando proyecto..."'
                // Si usas Composer: sh 'composer install --no-dev'
            }
        }

        stage('Security Scan') {
            agent any
            steps {
                script {
                    echo "🔍 Ejecutando análisis de seguridad con Semgrep..."
                    
                    // Ejecutar Semgrep
                    sh """
                        docker run --rm \
                            -v \${WORKSPACE}:/src \
                            -w /src \
                            ${env.SEMGREP_IMAGE} \
                            semgrep scan \
                            --config ${env.SEMGREP_RULES} \
                            --json \
                            --output semgrep-report.json \
                            --no-git-ignore \
                            --metrics=off \
                            .
                    """
                    
                    // Analizar resultados
                    def report = readJSON file: 'semgrep-report.json'
                    def findings = report.results ?: []
                    def scannedFiles = report.paths?.scanned ?: []
                    
                    echo "📊 Archivos escaneados: ${scannedFiles.size()}"
                    echo "🔍 Vulnerabilidades encontradas: ${findings.size()}"
                    
                    // Si no se escanearon archivos, fallar
                    if (scannedFiles.size() == 0) {
                        error("❌ No se escaneó ningún archivo. Verifica el contenido del workspace.")
                    }
                    
                    // Si hay vulnerabilidades, mostrarlas y fallar
                    if (findings.size() > 0) {
                        echo "\n⚠️  VULNERABILIDADES DETECTADAS:\n"
                        
                        findings.each { issue ->
                            def severity = issue.extra?.severity ?: 'UNKNOWN'
                            def icon = severity == 'ERROR' ? '🔴' : severity == 'WARNING' ? '🟠' : '🟡'
                            
                            echo "${icon} [${severity}] ${issue.check_id}"
                            echo "   📁 ${issue.path}:${issue.start?.line ?: '?'}"
                            echo "   💬 ${issue.extra?.message ?: 'Sin descripción'}\n"
                        }
                        
                        error("❌ Build fallido: ${findings.size()} problema(s) de seguridad detectados")
                    }
                    
                    echo "✅ No se encontraron vulnerabilidades"
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: 'semgrep-report.json', allowEmptyArchive: true
                }
            }
        }

        stage('Deploy') {
            agent {
                docker { 
                    image 'php:8.2-cli'
                    args '-u root'
                }
            }
            when {
                expression { currentBuild.result == null || currentBuild.result == 'SUCCESS' }
            }
            steps {
                echo "🚀 Desplegando aplicación..."
                sh 'echo "docker run -d my-php-app"'
            }
        }
    }
    
    post {
        failure {
            script {
                echo "❌ Pipeline fallido. Revisa el reporte de Semgrep en los artefactos."
            }
        }
        success {
            script {
                echo "✅ Pipeline completado exitosamente"
            }
        }
    }
}
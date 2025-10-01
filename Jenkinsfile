pipeline {
    agent any
    
    options {
        // Mantener solo los últimos 10 builds
        buildDiscarder(logRotator(numToKeepStr: '10'))
        // Timeout global del pipeline
        timeout(time: 30, unit: 'MINUTES')
        // No permitir builds concurrentes
        disableConcurrentBuilds()
        // Mostrar timestamps en los logs
        timestamps()
    }
    
    environment {
        // Configuración centralizada
        SEMGREP_IMAGE = 'returntocorp/semgrep:latest'
        SEMGREP_RULES = 'p/security-audit,p/owasp-top-ten,p/php'
        SEMGREP_REPORT = 'semgrep-report.json'
        PHP_IMAGE = 'php:8.2-cli'
    }
    
    stages {
        stage('Checkout') {
            steps {
                cleanWs() // Limpiar workspace antes de empezar
                checkout scm
                
                script {
                    // Información del build
                    echo "📦 Build #${env.BUILD_NUMBER}"
                    echo "🌿 Branch: ${env.GIT_BRANCH ?: 'N/A'}"
                    echo "📝 Commit: ${env.GIT_COMMIT?.take(7) ?: 'N/A'}"
                }
                
                sh '''
                    echo "📂 Archivos en el proyecto:"
                    echo "  PHP: $(find . -name '*.php' -type f | wc -l)"
                    echo "  JS:  $(find . -name '*.js' -type f | wc -l)"
                '''
            }
        }

        stage('Build & Test') {
            agent {
                docker {
                    image "${env.PHP_IMAGE}"
                    args '-u root'
                    reuseNode true // Reusar el workspace del checkout
                }
            }
            steps {
                sh '''
                    php --version
                    echo "✓ PHP environment ready"
                '''
                // Descomentar si usas Composer:
                // sh 'composer install --no-dev --optimize-autoloader'
                // sh 'composer test'
            }
        }

        stage('Security Scan') {
            steps {
                script {
                    echo "🔍 Iniciando análisis de seguridad..."
                    
                    // Ejecutar Semgrep con manejo de errores
                    def semgrepStatus = sh(
                        returnStatus: true,
                        script: """
                            docker run --rm \
                                -v "\${WORKSPACE}:/src" \
                                -w /src \
                                ${env.SEMGREP_IMAGE} \
                                semgrep scan \
                                --config ${env.SEMGREP_RULES} \
                                --json \
                                --output ${env.SEMGREP_REPORT} \
                                --metrics=off \
                                --quiet \
                                .
                        """
                    )
                    
                    // Validar que el reporte existe
                    if (!fileExists(env.SEMGREP_REPORT)) {
                        error("❌ No se generó el reporte de Semgrep")
                    }
                    
                    // Parsear y analizar resultados
                    analyzeSemgrepReport()
                }
            }
            post {
                always {
                    // Publicar reporte como artefacto
                    archiveArtifacts(
                        artifacts: env.SEMGREP_REPORT,
                        allowEmptyArchive: true,
                        fingerprint: true
                    )
                    
                    // Publicar en formato HTML si existe plugin
                    publishHTML([
                        allowMissing: true,
                        alwaysLinkToLastBuild: true,
                        keepAll: true,
                        reportDir: '.',
                        reportFiles: env.SEMGREP_REPORT,
                        reportName: 'Semgrep Security Report'
                    ])
                }
            }
        }

        stage('Deploy') {
            when {
                allOf {
                    expression { currentBuild.result == null }
                    branch 'main' // Solo en main branch
                }
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
                // sh 'docker build -t my-app:${BUILD_NUMBER} .'
                // sh 'docker push my-app:${BUILD_NUMBER}'
            }
        }
    }
    
    post {
        always {
            script {
                def duration = currentBuild.durationString.replace(' and counting', '')
                echo """
╔═══════════════════════════════════════════════════════════╗
║                  RESUMEN DEL PIPELINE                     ║
╠═══════════════════════════════════════════════════════════╣
║  Estado:   ${currentBuild.currentResult}
║  Duración: ${duration}
║  Build:    #${env.BUILD_NUMBER}
╚═══════════════════════════════════════════════════════════╝
                """
            }
        }
        success {
            echo "✅ Pipeline completado exitosamente"
        }
        failure {
            echo "❌ Pipeline fallido. Revisa los logs y el reporte de Semgrep."
        }
        unstable {
            echo "⚠️  Pipeline marcado como inestable"
        }
        cleanup {
            // Limpiar recursos de Docker si es necesario
            sh 'docker system prune -f || true'
        }
    }
}

// Función helper para analizar el reporte de Semgrep
def analyzeSemgrepReport() {
    def report = readJSON file: env.SEMGREP_REPORT
    def findings = report.results ?: []
    def scannedFiles = report.paths?.scanned ?: []
    
    echo """
╔═══════════════════════════════════════════════════════════╗
║              RESULTADOS DEL ANÁLISIS                      ║
╚═══════════════════════════════════════════════════════════╝
📊 Archivos escaneados: ${scannedFiles.size()}
🔍 Vulnerabilidades:    ${findings.size()}
    """
    
    // Validar que se escanearon archivos
    if (scannedFiles.size() == 0) {
        error("❌ No se escaneó ningún archivo")
    }
    
    // Si no hay vulnerabilidades, éxito
    if (findings.size() == 0) {
        echo "✅ No se encontraron vulnerabilidades"
        return
    }
    
    // Clasificar por severidad
    def bySeverity = findings.groupBy { it.extra?.severity ?: 'INFO' }
    def critical = bySeverity['ERROR']?.size() ?: 0
    def high = bySeverity['WARNING']?.size() ?: 0
    def medium = bySeverity['INFO']?.size() ?: 0
    
    echo """
📈 Por severidad:
   🔴 Crítico:  ${critical}
   🟠 Alto:     ${high}
   🟡 Medio:    ${medium}
    """
    
    // Mostrar top vulnerabilidades
    def topFindings = findings
        .sort { a, b -> 
            def severityOrder = ['ERROR': 0, 'WARNING': 1, 'INFO': 2]
            severityOrder[a.extra?.severity] <=> severityOrder[b.extra?.severity]
        }
        .take(10)
    
    echo "\n⚠️  TOP VULNERABILIDADES:\n"
    topFindings.eachWithIndex { issue, idx ->
        def severity = issue.extra?.severity ?: 'INFO'
        def icon = ['ERROR': '🔴', 'WARNING': '🟠', 'INFO': '🟡'][severity]
        def message = issue.extra?.message?.take(80) ?: 'Sin descripción'
        
        echo "${idx + 1}. ${icon} ${issue.check_id}"
        echo "   📁 ${issue.path}:${issue.start?.line ?: '?'}"
        echo "   💬 ${message}...\n"
    }
    
    if (findings.size() > 10) {
        echo "... y ${findings.size() - 10} más (ver reporte completo)\n"
    }
    
    // Decidir si fallar el build
    if (critical > 0) {
        error("❌ Build fallido: ${critical} vulnerabilidades críticas encontradas")
    } else if (high > 5) {
        // Marcar como unstable si hay más de 5 high
        currentBuild.result = 'UNSTABLE'
        echo "⚠️  Build marcado como UNSTABLE por ${high} vulnerabilidades de alta severidad"
    } else {
        echo "⚠️  ${findings.size()} vulnerabilidades encontradas (ninguna crítica)"
    }
}
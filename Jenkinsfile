pipeline {
    agent none
    
    environment {
        SEMGREP_IMAGE = 'returntocorp/semgrep:latest'
        // Puedes cambiar las reglas según necesites:
        // p/security-audit - Más completo
        // p/owasp-top-ten - OWASP Top 10
        // p/ci - Reglas optimizadas para CI/CD
        SEMGREP_RULES = 'p/security-audit'
    }
    
    stages {
        stage('Checkout') {
            agent {
                docker { 
                    image 'php:8.2-cli'
                    args '-u root'
                }
            }
            steps {
                sh 'php --version'
                echo "✓ Código fuente verificado"
            }
        }

        stage('Compilation') {
            agent {
                docker { 
                    image 'php:8.2-cli'
                    args '-u root'
                }
            }
            steps {
                sh 'echo "Compilando proyecto PHP..."'
                // Si usas Composer:
                // sh 'composer install --no-dev --optimize-autoloader'
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
                sh 'echo "Construyendo imagen Docker..."'
                // sh 'docker build -t my-php-app .'
            }
        }

        stage('Semgrep Security Scan') {
            agent any
            steps {
                script {
                    echo "🔍 Iniciando análisis de seguridad con Semgrep..."
                    echo "📋 Usando reglas: ${SEMGREP_RULES}"
                    
                    // Ejecutar Semgrep y capturar el código de salida
                    def semgrepExitCode = sh(
                        script: """
                            docker run --rm \
                                -v \${WORKSPACE}:/src:ro \
                                -v \${WORKSPACE}:/output \
                                ${SEMGREP_IMAGE} semgrep scan \
                                --config "${SEMGREP_RULES}" \
                                --json \
                                --output /output/semgrep-report.json \
                                --verbose \
                                /src
                        """,
                        returnStatus: true
                    )
                    
                    echo "Código de salida de Semgrep: ${semgrepExitCode}"
                    
                    // Verificar que el reporte existe
                    if (!fileExists('semgrep-report.json')) {
                        error("❌ Error: No se pudo generar el reporte de Semgrep")
                    }
                    
                    // Leer y analizar el reporte JSON
                    def report = readJSON file: 'semgrep-report.json'
                    def results = report.results ?: []
                    def errors = report.errors ?: []
                    
                    // Mostrar información general
                    echo """
╔═══════════════════════════════════════════════════════════╗
║           REPORTE DE ANÁLISIS DE SEGURIDAD                ║
╚═══════════════════════════════════════════════════════════╝
                    """
                    
                    echo "📊 Total de problemas encontrados: ${results.size()}"
                    
                    if (errors.size() > 0) {
                        echo "⚠️  Errores durante el escaneo: ${errors.size()}"
                    }
                    
                    // Clasificar por severidad
                    def criticalIssues = results.findAll { it.extra?.severity == 'ERROR' }
                    def highIssues = results.findAll { it.extra?.severity == 'WARNING' }
                    def mediumIssues = results.findAll { it.extra?.severity == 'INFO' }
                    
                    echo """
📈 Distribución por severidad:
   🔴 CRÍTICO (ERROR):   ${criticalIssues.size()}
   🟠 ALTO (WARNING):    ${highIssues.size()}
   🟡 MEDIO (INFO):      ${mediumIssues.size()}
                    """
                    
                    // Si hay resultados, mostrar detalles y fallar
                    if (results.size() > 0) {
                        echo "\n🔍 DETALLES DE VULNERABILIDADES ENCONTRADAS:\n"
                        
                        // Mostrar primero los críticos
                        if (criticalIssues.size() > 0) {
                            echo "═══════════════════════════════════════════════════════════"
                            echo "🔴 PROBLEMAS CRÍTICOS (${criticalIssues.size()})"
                            echo "═══════════════════════════════════════════════════════════"
                            criticalIssues.eachWithIndex { r, index ->
                                echo """
[${index + 1}] ${r.check_id}
    📁 Archivo: ${r.path}
    📍 Línea: ${r.start?.line ?: 'N/A'} - ${r.end?.line ?: 'N/A'}
    💬 ${r.extra?.message ?: 'Sin descripción'}
    🔗 Más info: ${r.extra?.metadata?.source ?: 'N/A'}
                                """.stripIndent()
                            }
                        }
                        
                        // Mostrar los de alta severidad
                        if (highIssues.size() > 0) {
                            echo "\n═══════════════════════════════════════════════════════════"
                            echo "🟠 PROBLEMAS DE ALTA SEVERIDAD (${highIssues.size()})"
                            echo "═══════════════════════════════════════════════════════════"
                            highIssues.eachWithIndex { r, index ->
                                echo """
[${index + 1}] ${r.check_id}
    📁 Archivo: ${r.path}
    📍 Línea: ${r.start?.line ?: 'N/A'} - ${r.end?.line ?: 'N/A'}
    💬 ${r.extra?.message ?: 'Sin descripción'}
                                """.stripIndent()
                            }
                        }
                        
                        // Mostrar resumen de los de severidad media (sin tanto detalle)
                        if (mediumIssues.size() > 0) {
                            echo "\n═══════════════════════════════════════════════════════════"
                            echo "🟡 PROBLEMAS DE SEVERIDAD MEDIA (${mediumIssues.size()})"
                            echo "═══════════════════════════════════════════════════════════"
                            mediumIssues.take(5).eachWithIndex { r, index ->
                                echo "[${index + 1}] ${r.check_id} en ${r.path}:${r.start?.line}"
                            }
                            if (mediumIssues.size() > 5) {
                                echo "... y ${mediumIssues.size() - 5} más (ver reporte completo)"
                            }
                        }
                        
                        echo "\n═══════════════════════════════════════════════════════════"
                        echo "❌ BUILD FALLIDO: Se encontraron ${results.size()} problema(s) de seguridad"
                        echo "📄 Reporte completo disponible en los artefactos del build"
                        echo "═══════════════════════════════════════════════════════════\n"
                        
                        error("Build detenido por problemas de seguridad detectados por Semgrep")
                    } else {
                        echo """
╔═══════════════════════════════════════════════════════════╗
║  ✅ ¡EXCELENTE! No se encontraron problemas de seguridad  ║
╚═══════════════════════════════════════════════════════════╝
                        """
                    }
                }
            }
            post {
                always {
                    // Archivar el reporte JSON siempre, incluso si falla
                    archiveArtifacts artifacts: 'semgrep-report.json', 
                                   allowEmptyArchive: true, 
                                   fingerprint: true
                    
                    echo "📄 Reporte guardado: semgrep-report.json"
                }
            }
        }

        stage('Generate SARIF Report') {
            agent any
            when {
                expression { fileExists('semgrep-report.json') }
            }
            steps {
                script {
                    echo "📝 Generando reporte en formato SARIF..."
                    sh """
                        docker run --rm \
                            -v \${WORKSPACE}:/src:ro \
                            -v \${WORKSPACE}:/output \
                            ${SEMGREP_IMAGE} semgrep scan \
                            --config "${SEMGREP_RULES}" \
                            --sarif \
                            --output /output/semgrep-report.sarif \
                            /src || true
                    """
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: 'semgrep-report.sarif', 
                                   allowEmptyArchive: true
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
                // Solo desplegar si no hay fallos en etapas anteriores
                expression { currentBuild.result == null || currentBuild.result == 'SUCCESS' }
            }
            steps {
                echo "🚀 Desplegando aplicación..."
                sh 'echo "docker run -d -p 8080:80 my-php-app"'
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
║  Estado: ${currentBuild.result ?: 'SUCCESS'}
║  Duración: ${currentBuild.durationString}
╚═══════════════════════════════════════════════════════════╝
                """
            }
        }
        failure {
            script {
                echo """
❌ PIPELINE FALLIDO
═══════════════════════════════════════════════════════════
Por favor revisa:
  1. Los logs de Semgrep arriba
  2. El archivo semgrep-report.json en los artefactos
  3. Corrige las vulnerabilidades antes de hacer merge/deploy
═══════════════════════════════════════════════════════════
                """
            }
        }
        success {
            script {
                echo """
✅ PIPELINE EXITOSO
═══════════════════════════════════════════════════════════
Todo en orden:
  ✓ Código analizado
  ✓ Sin vulnerabilidades detectadas
  ✓ Listo para despliegue
═══════════════════════════════════════════════════════════
                """
            }
        }
    }
}
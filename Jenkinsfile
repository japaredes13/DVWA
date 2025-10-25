pipeline {
    agent any
    
    environment {
        // Definir variables de entorno
        SEMGREP_VERSION = 'latest'
        REPORT_DIR = 'reports'
        SEMGREP_REPORT = 'semgrep-results.json'
    }
    
    stages {
        stage('Checkout') {
            steps {
                script {
                    echo '📦 Descargando código fuente de DVWA...'
                    // Si DVWA está en un repositorio Git
                    checkout scm
                    // O puedes clonar desde GitHub:
                    // git url: 'https://github.com/digininja/DVWA.git', branch: 'master'
                }
            }
        }
        
        stage('Preparar Entorno') {
            steps {
                script {
                    echo '🔧 Preparando entorno de análisis...'
                    // Crear directorio para reportes
                    sh "mkdir -p ${REPORT_DIR}"
                    
                    // Verificar versión de Python (Semgrep requiere Python)
                    sh 'python3 --version || python --version'
                }
            }
        }
        
        stage('Instalar Semgrep') {
            steps {
                script {
                    echo '⚙️ Instalando Semgrep...'
                    // Instalar Semgrep usando pip
                    sh '''
                        pip3 install semgrep || pip install semgrep
                        semgrep --version
                    '''
                }
            }
        }
        
        stage('Análisis SAST con Semgrep') {
            steps {
                script {
                    echo '🔍 Ejecutando análisis de seguridad con Semgrep...'
                    // Ejecutar Semgrep con reglas automáticas
                    sh """
                        semgrep scan \
                            --config=auto \
                            --json \
                            --output=${REPORT_DIR}/${SEMGREP_REPORT} \
                            . || true
                    """
                    
                    echo '✅ Análisis de Semgrep completado'
                }
            }
        }
        
        stage('Procesar Resultados') {
            steps {
                script {
                    echo '📊 Procesando resultados del análisis...'
                    
                    // Verificar si el archivo fue generado
                    sh """
                        if [ -f ${REPORT_DIR}/${SEMGREP_REPORT} ]; then
                            echo "✓ Reporte generado exitosamente"
                            echo "Ubicación: ${REPORT_DIR}/${SEMGREP_REPORT}"
                            
                            # Mostrar resumen de vulnerabilidades
                            echo "=== RESUMEN DE VULNERABILIDADES ==="
                            cat ${REPORT_DIR}/${SEMGREP_REPORT} | python3 -m json.tool | head -50
                        else
                            echo "✗ Error: No se generó el reporte"
                            exit 1
                        fi
                    """
                }
            }
        }
        
        stage('Archivar Resultados') {
            steps {
                script {
                    echo '💾 Archivando resultados...'
                    // Archivar el reporte JSON en Jenkins
                    archiveArtifacts artifacts: "${REPORT_DIR}/*.json", 
                                     fingerprint: true,
                                     allowEmptyArchive: false
                }
            }
        }
        
        stage('Evaluación de Seguridad') {
            steps {
                script {
                    echo '⚠️ Evaluando nivel de seguridad...'
                    
                    // Opcional: Fallar el build si hay vulnerabilidades críticas
                    sh """
                        # Contar vulnerabilidades por severidad
                        HIGH_VULN=\$(cat ${REPORT_DIR}/${SEMGREP_REPORT} | grep -o '"severity":"ERROR"' | wc -l || echo 0)
                        MEDIUM_VULN=\$(cat ${REPORT_DIR}/${SEMGREP_REPORT} | grep -o '"severity":"WARNING"' | wc -l || echo 0)
                        
                        echo "Vulnerabilidades CRÍTICAS: \$HIGH_VULN"
                        echo "Vulnerabilidades MEDIAS: \$MEDIUM_VULN"
                        
                        # Opcional: Descomentar para fallar el build si hay vulnerabilidades críticas
                        # if [ \$HIGH_VULN -gt 0 ]; then
                        #     echo "❌ Build fallido: Se encontraron \$HIGH_VULN vulnerabilidades críticas"
                        #     exit 1
                        # fi
                    """
                }
            }
        }
    }
    
    post {
        always {
            echo '🧹 Limpiando workspace...'
            // Limpiar archivos temporales si es necesario
        }
        success {
            echo '✅ Pipeline ejecutado exitosamente'
            echo "📄 Reporte disponible en: ${REPORT_DIR}/${SEMGREP_REPORT}"
        }
        failure {
            echo '❌ Pipeline falló. Revisa los logs para más detalles.'
        }
    }
}

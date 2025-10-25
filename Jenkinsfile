pipeline {
    agent any
    
    environment {
        REPORT_DIR = 'reports'
        SEMGREP_REPORT = 'semgrep-results.json'
    }
    
    stages {
        stage('Checkout') {
            steps {
                script {
                    echo '📦 Descargando código fuente de DVWA...'
                    checkout scm
                }
            }
        }
        
        stage('Preparar Entorno') {
            steps {
                script {
                    echo '🔧 Preparando entorno de análisis...'
                    sh "mkdir -p ${REPORT_DIR}"
                }
            }
        }
        
        stage('Análisis SAST con Semgrep') {
            agent {
                docker {
                    image 'semgrep/semgrep:latest'
                    args '-v ${WORKSPACE}:/src --entrypoint=""'
                    reuseNode true
                }
            }
            steps {
                script {
                    echo '🔍 Ejecutando análisis de seguridad con Semgrep...'
                    sh """
                        semgrep scan \
                            --config=auto \
                            --json \
                            --output=${REPORT_DIR}/${SEMGREP_REPORT} \
                            /src || true
                    """
                    echo '✅ Análisis de Semgrep completado'
                }
            }
        }
        
        stage('Procesar Resultados') {
            steps {
                script {
                    echo '📊 Procesando resultados del análisis...'
                    
                    sh """
                        if [ -f ${REPORT_DIR}/${SEMGREP_REPORT} ]; then
                            echo "✓ Reporte generado exitosamente"
                            echo "Ubicación: ${REPORT_DIR}/${SEMGREP_REPORT}"
                            echo "Tamaño del archivo:"
                            ls -lh ${REPORT_DIR}/${SEMGREP_REPORT}
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
                    
                    sh """
                        # Contar vulnerabilidades por severidad
                        HIGH_VULN=\$(grep -o '"severity":"ERROR"' ${REPORT_DIR}/${SEMGREP_REPORT} | wc -l || echo 0)
                        MEDIUM_VULN=\$(grep -o '"severity":"WARNING"' ${REPORT_DIR}/${SEMGREP_REPORT} | wc -l || echo 0)
                        
                        echo "Vulnerabilidades CRÍTICAS: \$HIGH_VULN"
                        echo "Vulnerabilidades MEDIAS: \$MEDIUM_VULN"
                        
                        # Opcional: Descomentar para fallar el build
                        # if [ \$HIGH_VULN -gt 0 ]; then
                        #     echo "❌ Build fallido: \$HIGH_VULN vulnerabilidades críticas"
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

pipeline {
    agent any

    environment {
        SEMGREP_IMAGE = 'returntocorp/semgrep:latest'
        SEMGREP_RULES = 'p/owasp-top-ten'  // podés cambiar por p/security-audit o p/ci
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Semgrep Security Scan') {
            steps {
                script {
                    // Ejecutar semgrep dentro del contenedor
                    def exitCode = sh(
                        script: """
                            docker run --rm \
                                -v \${WORKSPACE}:/src \
                                ${SEMGREP_IMAGE} semgrep scan \
                                --config "${SEMGREP_RULES}" \
                                --json --output /src/semgrep-report.json \
                                --error --metrics=off /src
                        """,
                        returnStatus: true
                    )

                    // Leer reporte JSON
                    def report = readJSON file: 'semgrep-report.json'
                    def results = report.results ?: []

                    echo "📊 Total de hallazgos: ${results.size()}"

                    // Mostrar resumen en consola
                    results.take(10).eachWithIndex { r, i ->
                        echo "[${i+1}] ${r.check_id} en ${r.path}:${r.start?.line} - ${r.extra?.message}"
                    }
                    if (results.size() > 10) {
                        echo "... y ${results.size() - 10} más (ver semgrep-report.json)"
                    }

                    // Si hay findings, falla el pipeline
                    if (results.size() > 0) {
                        error("❌ Se encontraron vulnerabilidades, el deploy no continuará.")
                    } else {
                        echo "✅ No se encontraron problemas de seguridad"
                    }
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: 'semgrep-report.json', allowEmptyArchive: true
                }
            }
        }

        stage('Build') {
            when {
                expression { currentBuild.result == null || currentBuild.result == 'SUCCESS' }
            }
            steps {
                echo "⚙️ Construyendo la aplicación..."
                sh 'docker build -t my-app .'
            }
        }

        stage('Deploy') {
            when {
                expression { currentBuild.result == null || currentBuild.result == 'SUCCESS' }
            }
            steps {
                echo "🚀 Desplegando aplicación..."
                sh 'docker run -d -p 8080:80 my-app'
            }
        }
    }

    post {
        failure {
            echo "❌ Pipeline fallido. Revisa los hallazgos de Semgrep antes de volver a ejecutar."
        }
        success {
            echo "✅ Pipeline exitoso. La aplicación fue desplegada."
        }
    }
}

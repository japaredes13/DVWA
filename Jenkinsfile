pipeline {
    agent any
    environment {
        SEMGREP_DIR = "${WORKSPACE}/semgrep-reports"
    }
    stages {
        stage('SAST-Semgrep') {
            steps {
                script{
                    sh 'apt-get update && apt-get install -qq -y git'
                    sh 'git config --global --add safe.directory $(pwd)'
                    sh 'pip install -q semgrep'

                    sh 'echo "Creando directorio para reportes..."'
                    sh 'mkdir -p $SEMGREP_DIR'

                    sh 'echo "🚀 Ejecutando análisis Semgrep..."'
                    try {
                        sh 'semgrep scan --config=auto --json --error . > $SEMGREP_DIR/semgrep.json || true' // con el flag --json-output generamos un reporte en formato json y con --error hacemos que semgrep devuelva un código de salida distinto de 0 si encuentra alguna vulnerabilidad
                        sh 'echo "✅ Archivo generado:"'
                        sh 'ls -lh semgrep.json || echo "No existe semgrep.json"'
                    }
                    catch (err) {                                        
                        unstable(message: "Findings found") // marcamos el build como inestable si semgrep encuentra vulnerabilidades o si queremos bloquearlo podemos usar "error" en lugar de "unstable"
                    }
                }
            }
        }        
        stage('Compilation') {
            agent {
                docker { image 'php:8.2-cli' }
            }
            steps {
                sh 'echo "Compilando..."'
            }
        }
        stage('Build') {
            agent {
                docker { image 'php:8.2-cli' }
            }
            steps {
                sh 'echo "docker build -t my-php-app ."'
            }
        }
        stage('Deploy') {
            agent {
                docker { image 'php:8.2-cli' }
            }
            steps {
                sh 'echo "docker run my-php-app ."'
            }
        }
    }
    post {
        always {
            sh 'ls -lh || true'
            archiveArtifacts artifacts: 'semgrep-reports/semgrep.json', fingerprint: true, onlyIfSuccessful: false // guardamos el reporte de semgrep como artefacto del build para que persista en Jenkins
        }
    }
}

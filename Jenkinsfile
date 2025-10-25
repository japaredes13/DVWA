pipeline {
    agent any
    environment {
        SEMGREP_FILE = "${WORKSPACE}/semgrep.json"
    }
    stages {
        stage('SAST-Semgrep') {
            steps {
                script {
                    docker.image('python:3.11-slim').inside("-u root -v ${env.WORKSPACE}:${env.WORKSPACE} -w ${env.WORKSPACE}") {
                        sh 'apt-get update && apt-get install -qq -y git'
                        sh 'git config --global --add safe.directory $(pwd)'
                        sh 'pip install -q semgrep'
                        try {
                            sh 'semgrep scan --json-output=${SEMGREP_FILE} --error . || true' // con el flag --json-output generamos un reporte en formato json y con --error hacemos que semgrep devuelva un código de salida distinto de 0 si encuentra alguna vulnerabilidad
                            sh 'ls -lah ${WORKSPACE}'
                        } catch (err) {                                        
                            unstable(message: "Findings found") // marcamos el build como inestable si semgrep encuentra vulnerabilidades o si queremos bloquearlo podemos usar "error" en lugar de "unstable"
                        }
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
            sh 'echo "📦 Archivando reporte Semgrep..."'
            sh 'ls -lah ${WORKSPACE}' // Verificación de archivos generados
            archiveArtifacts artifacts: 'semgrep.json', fingerprint: true, allowEmptyArchive: false // guardamos el reporte de semgrep como artefacto del build para que persista en Jenkins
        }
    }
}

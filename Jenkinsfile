pipeline {
    agent none
    stages {
        stage('Checkout') {
            agent {
                docker { image 'php:8.2-cli' }
            }
            steps {
                sh 'php --version'
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

        stage('Semgrep Security Scan') {
            agent any  // Necesario porque agent none no permite steps que requieren nodo
            steps {
                sh '''
                docker run --rm \
                    -v $WORKSPACE:/src \
                    returntocorp/semgrep semgrep \
                    --config "p/owasp-top-ten" \
                    --auto \
                    /src \
                    --json > $WORKSPACE/semgrep-report.json
                '''
                // Verifica si hay hallazgos y falla el pipeline si existen
                sh '''
                python3 - <<EOF
                    import json
                    import sys

                    report_file = "$WORKSPACE/semgrep-report.json"
                    with open(report_file) as f:
                        data = json.load(f)

                    if data.get("results"):
                        print("⚠️ Semgrep found issues, failing the build!")
                        for r in data["results"]:
                            print(f'{r.get("check_id")} | {r.get("path")} | {r.get("start")}-{r.get("end")} | {r.get("extra", {}).get("message")}')
                        sys.exit(1)
                    EOF
                '''
            }
        }

        stage('Publish Semgrep Report') {
            agent any
            steps {
                archiveArtifacts artifacts: 'semgrep-report.json', fingerprint: true
                sh 'echo "Semgrep report archived at $WORKSPACE/semgrep-report.json"'
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
}

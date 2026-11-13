// Jenkinsfile - Pipeline de "Sistema Alumnos" (OCY1102)
pipeline {
    agent any

    options {
        timestamps()
        timeout(time: 30, unit: 'MINUTES')
        disableConcurrentBuilds()
    }

    triggers {
        pollSCM('H/5 * * * *')
    }

    environment {
        PIPENV_VENV_IN_PROJECT    = '1'
        PIPENV_IGNORE_VIRTUALENVS = '1'
        PIPENV_NOSPIN             = '1'
    }

    stages {
        stage('Build') {
            steps {
                script {
                    def sha = sh(returnStdout: true, script: 'git rev-parse --short HEAD').trim()
                    env.GIT_SHA     = sha
                    env.APP_VERSION = "1.0.${env.BUILD_NUMBER}+${sha}"
                }
                echo "Versión de este build: ${env.APP_VERSION}"
                sh 'rm -rf .venv reports && mkdir -p reports'
                sh 'pipenv install --dev --deploy'
                sh 'pipenv run python -m compileall -q app.py'
            }
        }

        stage('Test') {
            steps {
                sh 'pipenv run pytest --junitxml=reports/junit.xml'
            }
            post {
                always {
                    junit allowEmptyResults: true, testResults: 'reports/junit.xml'
                }
            }
        }

        stage('Generate Documentation') {
            steps {
                sh 'doxygen Doxyfile 2> reports/doxygen_warnings.log || true'
                sh 'test -f docs/html/index.html'
            }
        }

        stage('Version Control') {
            steps {
                sh 'chmod +x scripts/trazabilidad.sh && ./scripts/trazabilidad.sh'
                sh '''
                    git config user.email "jenkins@local.dev"
                    git config user.name "Jenkins CI"
                    git add -f docs/
                    if ! git diff --cached --quiet; then
                        git commit -m "docs: actualiza documentacion y trazabilidad (${APP_VERSION}) [skip ci]"
                    else
                        echo "Sin cambios nuevos en docs/"
                    fi
                '''
            }
        }

        stage('Deploy') {
            steps {
                sh 'chmod +x scripts/deploy_test.sh && ./scripts/deploy_test.sh'
            }
        }
    }

    post {
        always {
            archiveArtifacts artifacts: 'reports/**, docs/TRAZABILIDAD.md', allowEmptyArchive: true
        }
        unstable {
            script {
                notificar('UNSTABLE')
            }
        }
        failure {
            script {
                notificar('FAILURE')
            }
        }
    }
}

def notificar(String estado) {
    echo "ALERTA [${estado}]: El pipeline ${env.JOB_NAME} #${env.BUILD_NUMBER} requiere revisión (${env.BUILD_URL})."
}
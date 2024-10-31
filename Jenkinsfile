pipeline {
    agent any
    environment {
        SCM_URL = "https://github.com/Fprietod/testing_jenkins.git"
    }
    stages {
        stage('Checkout PR') {
            steps {
                script {
                    def branchToBuild = env.CHANGE_ID ? "refs/pull/${env.CHANGE_ID}/head" : "refs/heads/${env.BRANCH_NAME}"
                    
                    if (!env.CHANGE_ID && !env.BRANCH_NAME) {
                        error "No se pudo determinar la rama para el checkout. Verifica que el pipeline esté en el contexto correcto."
                    }

                    checkout([$class: 'GitSCM',
                              branches: [[name: branchToBuild]],
                              userRemoteConfigs: [[url: SCM_URL, credentialsId: 'github-credentials-id']],
                              extensions: [[$class: 'CleanBeforeCheckout']]
                    ])
                }
            }
        }
        stage('SQLFluff Analysis') {
            steps {
                script {
                    def lintResult = sh(script: 'sqlfluff lint **/*.sql --dialect mysql --rules L001,L002,L003,L004', returnStatus: true)
                    if (lintResult != 0) {
                        if (env.CHANGE_ID) {
                            withCredentials([string(credentialsId: 'github-token', variable: 'GITHUB_TOKEN')]) {
                                sh """
                                curl -X PATCH -H "Authorization: token ${GITHUB_TOKEN}" -H "Accept: application/vnd.github.v3+json" \
                                -d '{"state": "closed"}' \
                                https://api.github.com/repos/Fprietod/testing_jenkins/pulls/${env.CHANGE_ID}
                                """
                            }
                        } else {
                            echo "No es un contexto de PR. No se puede cerrar el PR."
                        }
                        error "La validación de calidad no pasó. PR cerrado."
                    }
                }
            }
        }
        stage('Approve PR if Quality Check Passes') {
            steps {
                script {
                    if (env.CHANGE_ID) {
                        withCredentials([string(credentialsId: 'github-token', variable: 'GITHUB_TOKEN')]) {
                            sh """
                            curl -X POST -H "Authorization: token ${GITHUB_TOKEN}" -H "Accept: application/vnd.github.v3+json" \
                            -d '{"event": "APPROVE"}' \
                            https://api.github.com/repos/Fprietod/testing_jenkins/pulls/${env.CHANGE_ID}/reviews
                            """
                        }
                    } else {
                        echo "No es un contexto de PR. No se puede aprobar el PR."
                    }
                }
            }
        }
    }
    post {
        success {
            echo 'Pipeline ejecutado con éxito. PR aprobado.'
        }
        failure {
            echo 'El pipeline falló. Revisa los errores de calidad del código SQL.'
        }
    }
}

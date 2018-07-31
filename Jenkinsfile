#!groovy
library 'jenkins-pipeline-shared'

pipeline {
    environment {
        MODULE_NAME = "zipkin-server"
        FAILED_STAGE = "NONE"
        BI = "bi"
        SBR = "sbr"
    }
    options {
        skipDefaultCheckout()
        buildDiscarder(logRotator(numToKeepStr: '30', artifactNumToKeepStr: '30'))
        timeout(time: 30, unit: 'MINUTES')
        timestamps()
    }
    agent any
    stages {
        stage('Checkout') {
            agent any
            steps {
                deleteDir()
                checkout scm
                script {
                    def server = Artifactory.server 'art-p-01'
                    def buildInfo = Artifactory.newBuildInfo()
                    def downloadSpec = readFile 'resources/download.json'
                    server.download spec: downloadSpec, buildInfo: buildInfo
                }
                sh "cp ${MODULE_NAME}/${MODULE_NAME}*.jar ${MODULE_NAME}.jar"
            }
            post {
                success {
                    stageSuccess()
                }
                failure {
                    stageFailure()
                }
            }
        }

        stage('Test'){
            agent any
            steps {
                script {
                    def exists = fileExists "${MODULE_NAME}.jar"
                    if (exists) {
                        stageSuccess()
                    } else {
                        stageFailure()
                        error("Cannot find file ${MODULE_NAME}.jar")
                    }
                }
            }
        }

        stage('Deploy - DEV') {
            environment {
                DEPLOY_TO = "dev"
                MANIFEST_DIR = "resources"
            }
            parallel {
                stage('BI') {
                    agent any
                    environment {
                        TEAM = ${BI}
                        CF_ROUTE = "${DEPLOY_TO}-${TEAM}-${MODULE_NAME}"
                    }
                    steps {
                        milestone(1)
                        lock("${CF_ROUTE}") {
                            deploy()
                        }
                    }
                }
                stage('SBR') {
                    agent any
                    environment {
                        TEAM = ${SBR}
                        CF_ROUTE = "${DEPLOY_TO}-${TEAM}-${MODULE_NAME}"
                    }
                    steps {
                        milestone(1)
                        lock("${CF_ROUTE}") {
                            deploy()
                        }
                    }
                }
            }
            post {
                success {
                    stageSuccess()
                }
                failure {
                    stageFailure()
                }
            }
        }

        stage('Test - DEV') {
            environment {
                DEPLOY_TO = "dev"
            }
            parallel {
                stage('BI') {
                    agent any
                    environment {
                        TEAM = ${BI}
                    }
                    steps {
                        healthCheck()
                    }
                }
                stage('SBR') {
                    agent any
                    environment {
                        TEAM = ${SBR}
                    }
                    steps {
                        healthCheck()
                    }
                }
            }
            post {
                success {
                    stageSuccess()
                }
                failure {
                    stageFailure()
                }
            }
        }
    }
    post {
        always {
            script {
                colourText("info", 'Post steps initiated')
                deleteDir()
            }
        }
        success {
            colourText("success", "All stages complete. Build was successful.")
            sendNotifications currentBuild.result, "\$SBR_EMAIL_LIST"
        }
        unstable {
            colourText("warn", "Something went wrong, build finished with result ${currentResult}. This may be caused by failed tests, code violation or in some cases unexpected interrupt.")
            sendNotifications currentBuild.result, "\$SBR_EMAIL_LIST", "${STAGE_NAME}"
        }
        failure {
            colourText("warn","Process failed at: ${FAILED_STAGE}")
            sendNotifications currentBuild.result, "\$SBR_EMAIL_LIST", "${FAILED_STAGE}"
        }
    }
}

def stageSuccess() {
    colourText("info", "Stage: ${STAGE_NAME} successful!")
}

def stageFailure() {
    FAILED_STAGE = ${env.STAGE_NAME}
    colourText("warn", "Stage: ${STAGE_NAME} failed!")
}


def deploy() {
    CF_SPACE = "${DEPLOY_TO}".capitalize()
    CF_ORG = "${TEAM}".toUpperCase()
    echo "Deploying app to ${TEAM} ${DEPLOY_TO}"
    deployToCloudFoundry("${TEAM}-${DEPLOY_TO}-cf", "${CF_ORG}", "${CF_SPACE}", "${CF_ROUTE}", "${MODULE_NAME}.jar", "${MANIFEST_DIR}/manifest.yml")
}

def healthCheck() {
    CF_URL = "http://${DEPLOY_TO}-${TEAM}-${MODULE_NAME}.${CLOUD_FOUNDRY_ROUTE_SUFFIX}"
    httpRequest consoleLogResponseBody: true, contentType: 'APPLICATION_JSON', responseHandle: 'NONE', url: "${CF_URL}/health", validResponseCodes: '200', validResponseContent: '{"zipkin":{"status":"UP","details":{"InMemoryStorage":{"status":"UP"}}},"status":"UP"}'

}

#!groovy
library 'jenkins-pipeline-shared'

pipeline {
    environment {
        TEAM = "sbr"
        MODULE_NAME = "zipkin-server"
        STAGE_NAME = "NONE";
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
                def exists = fileExists "${MODULE_NAME}.jar"
                if (exists) {
                    stageSuccess()
                } else {
                    stageFailure()
                    error("Cannot find file ${MODULE_NAME}.jar")
                }
            }
        }

        stage('Deploy - DEV') {
            agent any
            //when{ expression{ isBranch("master") }}
            environment {
                DEPLOY_TO = "dev"
                CF_ROUTE = "${DEPLOY_TO}-${TEAM}-${MODULE_NAME}"
                MANIFEST_DIR = "resources"
            }
            steps {
                milestone(1)
                lock("${env.CF_ROUTE}") {
                    deploy()
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

        stage('Test - DEV'){
            agent any
            steps {
                healthCheck()
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
            colourText("warn","Process failed at: ${STAGE_NAME}")
            sendNotifications currentBuild.result, "\$SBR_EMAIL_LIST", "${STAGE_NAME}"
        }
    }
}

def stageSuccess() {
    colourText("info", "Stage: ${env.STAGE_NAME} successful!")
}

def stageFailure() {
    STAGE_NAME = ${env.STAGE_NAME}
    colourText("warn", "Stage: ${env.STAGE_NAME} failed!")
}


def deploy() {
    CF_SPACE = "${env.DEPLOY_TO}".capitalize()
    CF_ORG = "${TEAM}".toUpperCase()
    echo "Deploying app to ${env.DEPLOY_TO}"
    deployToCloudFoundry("${TEAM}-${env.DEPLOY_TO}-cf", "${CF_ORG}", "${CF_SPACE}", "${env.CF_ROUTE}", "${MODULE_NAME}.jar", "${env.MANIFEST_DIR}/manifest.yml")
}

def healthCheck() {
    httpRequest consoleLogResponseBody: true, contentType: 'APPLICATION_JSON', responseHandle: 'NONE', url: "http://${CF_ROUTE}.${CLOUD_FOUNDRY_ROUTE_SUFFIX}/health", validResponseCodes: '200', validResponseContent: '{"zipkin":{"status":"UP","details":{"InMemoryStorage":{"status":"UP"}}},"status":"UP"}'

}

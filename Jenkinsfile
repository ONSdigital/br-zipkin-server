#!groovy
library 'jenkins-pipeline-shared'

pipeline {
    environment {
        TEAM = "sbr"
        MODULE_NAME = "zipkin-server"
        MANIFEST_DIR = "resources"
        DEV = "dev"
        TEST = "test"
        PROD = "beta"
        FAILED_STAGE = "NONE"
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
            agent any
            //when{ expression{ isBranch("master") }}
            environment {
                DEV_ROUTE = "${DEV}-${TEAM}-${MODULE_NAME}"
            }
            steps {
                milestone(1)
                lock("${env.DEV_ROUTE}") {
                    deploy("${TEAM}", "${DEV}", "${env.DEV_ROUTE}")
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
            agent any
            steps {
                healthCheck("${DEV}-${TEAM}-${MODULE_NAME}")
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
    
        stage('Deploy - TEST') {
            agent any
            //when{ expression{ isBranch("master") }}
            environment {
                TEST_ROUTE = "${TEST}-${TEAM}-${MODULE_NAME}"
            }
            steps {
                milestone(2)
                lock("${env.TEST_ROUTE}") {
					deploy("${TEAM}", "${TEST}", "${env.TEST_ROUTE}")
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

        stage('Test - TEST') {
            agent any
            steps {
                healthCheck("${TEST}-${TEAM}-${MODULE_NAME}")
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

        stage('Deploy - PROD') {
            agent any
            //when{ expression{ isBranch("master") }}
            environment {
                PROD_ROUTE = "${TEAM}-${MODULE_NAME}"
            }
            steps {
                milestone(3)
                lock("${env.PROD_ROUTE}") {
                    deploy("${TEAM}", "${PROD}", "${PROD_ROUTE}")
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

        stage('Test - PROD') {
            agent any
            steps {
                healthCheck("${TEAM}-${MODULE_NAME}")
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
    FAILED_STAGE = "${env.STAGE_NAME}"
    colourText("warn", "Stage: ${STAGE_NAME} failed!")
}

def deploy(String org, String space, String route) {
    echo "Deploying app to ${space}"
    deployToCloudFoundry("${org}-${space}-cf", "${org}".capitalize(), "${space}".capitalize(), "${route}", "${MODULE_NAME}.jar", "${MANIFEST_DIR}/manifest.yml")
}

def healthCheck(String route) {
    CF_URL = "http://${route}.${CLOUD_FOUNDRY_ROUTE_SUFFIX}"
    httpRequest consoleLogResponseBody: true, contentType: 'APPLICATION_JSON', responseHandle: 'NONE', url: "${CF_URL}/health", validResponseCodes: '200', validResponseContent: '{"zipkin":{"status":"UP","details":{"InMemoryStorage":{"status":"UP"}}},"status":"UP"}'
}

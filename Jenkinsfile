#!groovy
library 'jenkins-pipeline-shared'

pipeline {
    environment {
        ORGANIZATION = "ons"
        TEAM = "sbr"
        MODULE_NAME = "zipkin-server"

        SBT_HOME = tool name: 'sbt.13.13', type: 'org.jvnet.hudson.plugins.SbtPluginBuilder$SbtInstallation'
        PATH = "${env.SBT_HOME}/bin:${env.PATH}"
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
                stash name: 'app'
            }
        }
        stage('Download') {
            steps {
                script {
                    def server = Artifactory.server 'art-p-01'
                    def downloadSpec = readFile 'resources/download.json'
                    server.download spec: downloadSpec
                }
            }
            post {
                success {
                    colourText("info", "Stage: ${env.STAGE_NAME} successful!")
                }
                failure {
                    colourText("warn", "Stage: ${env.STAGE_NAME} failed!")
                }
            }
        }
        stage('Deploy - DEV') {
            agent any
            //when{ expression{ isBranch("master") }}
            environment {
                DEPLOY_TO = "dev"
                CF_ROUTE = "${env.DEPLOY_TO}-${MODULE_NAME}"
            }
            steps {
                milestone(1)
                lock("${env.CF_ROUTE}") {
                    colourText("info", "${CF_ROUTE} deployment in progress")
                    deploy()
                    colourText("success", "${CF_ROUTE} deployed")
                }
            }
            post {
                success {
                    colourText("info", "Stage: ${env.STAGE_NAME} successful!")
                }
                failure {
                    colourText("warn", "Stage: ${env.STAGE_NAME} failed!")
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
            colourText("warn","Process failed at: ${env.NODE_STAGE}")
            sendNotifications currentBuild.result, "\$SBR_EMAIL_LIST", "${STAGE_NAME}"
        }
    }
}

def deploy () {
    CF_SPACE = "${env.DEPLOY_TO}".capitalize()
    CF_ORG = "${TEAM}".toUpperCase()
    echo "Deploying app to ${env.DEPLOY_TO}"
    deployToCloudFoundry("${TEAM}-${env.DEPLOY_TO}-cf", "${CF_ORG}", "${CF_SPACE}", "${env.CF_ROUTE}", "${env.DEPLOY_TO}-${ORGANIZATION}-${MODULE_NAME}.zip", "gitlab/${env.DEPLOY_TO}/manifest.yml")
}

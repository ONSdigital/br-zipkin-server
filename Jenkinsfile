#!groovy
library 'jenkins-pipeline-shared'

// Global scope required for multi-stage persistence
def artServer = Artifactory.server 'art-p-01'
def buildInfo = Artifactory.newBuildInfo()
def agentMavenVersion = 'maven_3.5.4'
def downloadSpec

pipeline {
    environment {
        TEAM = "br"
        MODULE_NAME = "zipkin-server"
	SVC_NAME = "${TEAM}-${MODULE_NAME}"
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
    agent { label 'download.jenkins.slave' }
    stages {
	stage('Checkout') {
            agent { label 'download.jenkins.slave' }
            steps {
                checkout scm
                script {
                    buildInfo.name = "${SVC_NAME}"
                    buildInfo.number = "${BUILD_NUMBER}"
                    buildInfo.env.collect()
                }
                colourText("info", "BuildInfo: ${buildInfo.name}-${buildInfo.number}")
                dir('test-data') {
                    git branch: 'master', url: 'https://github.com/ONSdigital/br-zipkin-server.git'
                }
                stash name: 'Checkout'
            }
        }

        stage('Validate'){
            agent { label "build.${agentMavenVersion}" }
            steps {
		unstash name: 'Checkout'
                script {
                    downloadSpec = readFile 'resources/download.json'
                    server.download spec: downloadSpec, buildInfo: buildInfo
                }
                sh "cp ${MODULE_NAME}/${MODULE_NAME}*.jar ${MODULE_NAME}.jar"
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
            agent { label 'deploy.cf' }
            when { branch 'master' }
            environment {
                DEV_ROUTE = "${DEV}-${TEAM}-${MODULE_NAME}"
            }
            steps {
		milestone(1)
		script {
                    server.download spec: downloadSpec, buildInfo: buildInfo
                }
                sh "cp ${MODULE_NAME}/${MODULE_NAME}*.jar ${MODULE_NAME}.jar"
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
            agent { label 'deploy.cf' }
	        when { branch 'master' }
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
            agent { label 'deploy.cf' }
            when { branch 'master' }
            environment {
                TEST_ROUTE = "${TEST}-${TEAM}-${MODULE_NAME}"
            }
            steps {
                milestone(2)
		script {
                    server.download spec: downloadSpec, buildInfo: buildInfo
                }
                sh "cp ${MODULE_NAME}/${MODULE_NAME}*.jar ${MODULE_NAME}.jar"
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
            agent { label 'deploy.cf' }
    	    when { branch 'master' }
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
            agent { label 'deploy.cf' }
            when { branch 'master' }
            environment {
                PROD_ROUTE = "${TEAM}-${MODULE_NAME}"
            }
            steps {
                milestone(3)
                script {
                    server.download spec: downloadSpec, buildInfo: buildInfo
                }
                sh "cp ${MODULE_NAME}/${MODULE_NAME}*.jar ${MODULE_NAME}.jar"
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
            agent { label 'deploy.cf' }
    	    when { branch 'master' }
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
        success {
            colourText("success", "All stages complete. Build was successful.")
            slackSend(
                    color: "good",
                    message: "${env.JOB_NAME} success: ${env.RUN_DISPLAY_URL}"
            )
        }
        unstable {
            colourText("warn", "Something went wrong, build finished with result ${currentResult}. This may be caused by failed tests, code violation or in some cases unexpected interrupt.")
            slackSend(
                    color: "warning",
                    message: "${env.JOB_NAME} unstable: ${env.RUN_DISPLAY_URL}"
            )
        }
        failure {
            colourText("warn", "Process failed at: ${env.NODE_STAGE}")
            slackSend(
                    color: "danger",
                    message: "${env.JOB_NAME} failed at ${env.STAGE_NAME}: ${env.RUN_DISPLAY_URL}"
            )
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

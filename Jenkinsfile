#!groovy
def BUILD_NUMBER=env.BUILD_NUMBER
def ARTIFACT_DIR = "Builds\\${BUILD_NUMBER}"
def SFDC_USERNAME = ""
def HUB_ORG = env.HUB_ORG_DH
def SFDC_HOST = env.SFDC_HOST_DH
def JWT_KEY_CRED_ID = env.JWT_CRED_ID_DH
def CONNECTED_APP_CONSUMER_KEY = env.CONNECTED_APP_CONSUMER_KEY_DH

def toolbelt = tool 'toolbelt'

pipeline {
    agent any

    options {
        timestamps()
    }

    environment {

    }

    tools {
        toolbelt 'toolbelt'
    }

    stages {
        stage('Checkout SCM') {
            steps {
                checkout scm
            }
        }

        stage('Authorize DEV HUB org') {
            steps {
                withCredentials([file(credentialsId: JWT_KEY_CRED_ID, variable: 'jwt_key_file')]) {
                    status = bat(returnStatus: true, script: "\"${toolbelt}\" force:auth:jwt:grant --clientid ${CONNECTED_APP_CONSUMER_KEY} --username ${HUB_ORG} --jwtkeyfile \"${jwt_key_file}\" --setdefaultdevhubusername --instanceurl ${SFDC_HOST}")
                    if (status != 0) {
                        error 'Authorize DEV HUB org failed'
                    }
                }
            }
        }

        stage('Create SCRATCH org') {
            steps {
                withCredentials([file(credentialsId: JWT_KEY_CRED_ID, variable: 'jwt_key_file')]) {
                    stdout = bat(returnStdout: true, script: "\"${toolbelt}\" force:org:create --definitionfile config/project-scratch-def.json --json --setdefaultusername").trim()
                    stdout = stdout.readLines().drop(1).join(" ")
                    def robj = readJSON text: result;
                    if (robj.status != 0) {
                        error 'Create SCRATCH org failed: ' + robj.message
                    }
                    SFDC_USERNAME=robj.result.username
                    robj = null
                }
            }
        }

        stage('Push to SCRATCH org') {
            steps {
                withCredentials([file(credentialsId: JWT_KEY_CRED_ID, variable: 'jwt_key_file')]) {
                    status = bat(returnStatus: true, script: "\"${toolbelt}\" force:source:push --targetusername ${SFDC_USERNAME}")
                    if (status != 0) {
                            error 'push failed'
                    }
                    status = bat(returnStatus: true, script: "\"${toolbelt}\" force:user:permset:assign --targetusername ${SFDC_USERNAME} --permsetname ELS")
                    if (status != 0) {
                        error 'permset:assign failed'
                    }
                }
            }
        }

        stage('Run APEX tests') {
            steps {
                withCredentials([file(credentialsId: JWT_KEY_CRED_ID, variable: 'jwt_key_file')]) {
                    status = bat(returnStatus: true, script: "mkdir ${ARTIFACT_DIR}")
                    if(status != 0) {
                        error "Run APEX tests failed: cannot create artifact dir ${ARTIFACT_DIR}"
                    }
                    timeout(time: 120, unit: 'SECONDS') {
                        rc = bat(returnStatus: true, script: "\"${toolbelt}\" force:apex:test:run --testlevel RunLocalTests --outputdir ${ARTIFACT_DIR} --resultformat tap --targetusername ${SFDC_USERNAME}")
                        if (rc != 0) {
                            error 'Run APEX tests failed'
                        }
                    }
                }
            }
        }

        stage('Deploy to STAGING') {
            steps {
                echo 'Deploying...'
            }
        }

        stage('Collect Reports') {
            steps {
                junit keepLongStdio: true, testResults: 'tests/**/*-junit.xml'
            }
        }
    }
    post {
        always {
            status = bat(returnStatus: true, script: "\"${toolbelt}\" force:org:delete --targetusername ${SFDC_USERNAME} -p")
            if (rc != 0) { 
                echo 'Cleanup SCRATCH org failed'
            }
        }
        failure {
            // notify users when the Pipeline fails
            mail to: 'trinh.dang@sioux.asia',
            subject: "Failed Pipeline: ${currentBuild.fullDisplayName}",
            body: "Something is wrong with ${env.BUILD_URL}"
        }
    }
}

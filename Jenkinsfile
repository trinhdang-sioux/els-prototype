#!groovy

pipeline {
    agent any

    options {
        timestamps()
    }

    environment {
        ARTIFACT_DIR = "Builds\\${env.BUILD_NUMBER}"

        SFDC_USERNAME = ""
        HUB_ORG = "${env.HUB_ORG_DH}"
        SFDC_HOST = "${env.SFDC_HOST_DH}"
        CONNECTED_APP_CONSUMER_KEY = "${env.CONNECTED_APP_CONSUMER_KEY_DH}"
        CONNECTED_APP_JWT_KEY = credentials("${env.JWT_CRED_ID_DH}")

        sfdx = "C:\\Program Files\\Salesforce CLI\\bin\\sfdx"
    }

    stages {
        stage('Checkout SCM') {
            steps {
                checkout scm
            }
        }

        stage('Build and test') {
            stages {
                stage('Authorize DEV HUB org') {
                    steps {
                        echo 'Authorizing...'
                        script {
                            status = bat(returnStatus: true, script: "\"${sfdx}\" force:auth:jwt:grant --clientid ${CONNECTED_APP_CONSUMER_KEY} --username ${HUB_ORG} --jwtkeyfile \"${CONNECTED_APP_JWT_KEY}\" --setdefaultdevhubusername --instanceurl ${SFDC_HOST}")
                            if (status != 0) {
                                error 'Authorize DEV HUB org failed'
                            }
                        }
                    }
                }

                stage('Create SCRATCH org') {
                    steps {
                        echo 'Creating...'
                        script {
                            stdout = bat(returnStdout: true, script: "\"${sfdx}\" force:org:create --definitionfile config/project-scratch-def.json --json --setdefaultusername").trim()
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
                        echo 'Push to SCRATCH org'
                    }
                }

                stage('Run APEX tests') {
                    steps {
                        echo 'Run APEX tests'
                    }
                }
            }
            post {
                always {
                    echo 'Cleaning up...'
                    script {
                        status = bat(returnStatus: true, script: "\"${sfdx}\" force:org:delete --targetusername ${SFDC_USERNAME} -p")
                        if (status != 0) { 
                            echo 'Cleanup failed'
                        }
                    }
                }
            }
        }

        stage('Deploy') {
            steps {
                echo 'Deploy here'
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
            echo 'Post always'
        }
        failure {
            echo 'Email'
        }
    }
}

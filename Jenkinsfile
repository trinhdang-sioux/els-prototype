#!groovy

pipeline {
    agent any

    options {
        timestamps()
    }

    environment {
        BUILD_NUMBER = "${env.BUILD_NUMBER}"
        COMMIT_NUMBER = ""

        OUTPUT_DIR = "Builds\\${BUILD_NUMBER}"
        OUTPUT_TEST = "${OUTPUT_DIR}\\tests"
        OUTPUT_ARTIFACT = "${OUTPUT_DIR}\\artifacts"

        HUB_ORG = "${env.HUB_ORG_DH}"
        SFDC_HOST = "${env.SFDC_HOST_DH}"
        CONNECTED_APP_CONSUMER_KEY = "${env.CONNECTED_APP_CONSUMER_KEY_DH}"
        CONNECTED_APP_JWT_KEY = credentials("${env.JWT_CRED_ID_DH}")
        SFDC_USERNAME = ""

        sfdx = "C:\\Program Files\\Salesforce CLI\\bin\\sfdx"
    }

    stages {
        stage('checkout SCM') {
            steps {
                script {
                    scmVars = checkout scm
                    COMMIT_NUMBER = scmVars.GIT_COMMIT
                }
            }
        }

        stage('create output dir') {
            steps {
                script {
                    bat returnStatus: true, script: "mkdir ${OUTPUT_TEST}"
                    bat returnStatus: true, script: "mkdir ${OUTPUT_ARTIFACT}"
                }
            }
        }

        stage('build and test') {
            stages {
                stage('authorize dev hub org') {
                    steps {
                        script {
                            status = bat returnStatus: true, script: "\"${sfdx}\" force:auth:jwt:grant --clientid ${CONNECTED_APP_CONSUMER_KEY} --username ${HUB_ORG} --jwtkeyfile \"${CONNECTED_APP_JWT_KEY}\" --setdefaultdevhubusername --instanceurl ${SFDC_HOST}"
                            if (status != 0) {
                                error 'Org authorization failed'
                            }
                        }
                    }
                }

                stage('create scratch org') {
                    steps {
                        script {
                            stdout = bat returnStdout: true, script: "\"${sfdx}\" force:org:create --definitionfile config/project-scratch-def.json --json --setdefaultusername"
                            stdout = stdout.trim().readLines().drop(1).join(" ")
                            def robj = readJSON text: stdout;
                            if (robj.status != 0) {
                                error 'Org creation failed: ' + robj.message
                            }
                            SFDC_USERNAME = robj.result.username
                            robj = null
                        }
                    }
                }

                stage('push source to scratch org') {
                    steps {
                        script {
                            status = bat returnStatus: true, script: "\"${sfdx}\" force:source:push --targetusername ${SFDC_USERNAME}"
                            if (status != 0) {
                                    error 'Org push failed'
                            }
                            status = bat returnStatus: true, script: "\"${sfdx}\" force:user:permset:assign --targetusername ${SFDC_USERNAME} --permsetname ELS"
                            if (status != 0) {
                                error 'Org permset:assign failed'
                            }
                        }
                    }
                }

                stage('run apex tests') {
                    steps {
                        script {
                            timeout(time: 120, unit: 'SECONDS') {
                                rc = bat returnStatus: true, script: "\"${sfdx}\" force:apex:test:run --testlevel RunLocalTests --outputdir ${OUTPUT_TEST} --resultformat tap --targetusername ${SFDC_USERNAME}"
                                if (rc != 0) {
                                    error 'Run APEX tests failed'
                                }
                            }
                        }
                    }
                }
            }
            post {
                always {
                    script {
                        status = bat returnStatus: true, script: "\"${sfdx}\" force:org:delete --targetusername ${SFDC_USERNAME} -p"
                        if (status != 0) { 
                            error 'Cleanup failed'
                        }
                    }
                }
            }
        }

        stage('package') {
            environment {
                PACKAGE_DIR = "mdapi_output_dir"
                PACKAGE_NAME = "els-protoype"
                PACKAGE_ZIP = "${OUTPUT_ARTIFACT}\\${COMMIT_NUMBER}.${BUILD_NUMBER}.zip"
            }
            steps {
                script {
                    status = bat returnStatus: true, script: "sfdx force:source:convert -d ${PACKAGE_DIR}/ --packagename ${PACKAGE_NAME}"
                    if(status != 0) {
                        error 'package failed'
                    }
                    bat script: "dir /s /b mdapi_output_dir"
                    zip zipFile: "${PACKAGE_ZIP}", archive: false, dir: "${PACKAGE_DIR}"
                }
            }
        }

        stage('deploy to staging') {
            steps {
                echo 'Deploying to STAGING...'
                //TODO mdapi:deploy
            }
        }

        stage('collect reports') {
            steps {
                archiveArtifacts artifacts: "${OUTPUT_ARTIFACT}/*.*", fingerprint: true
                junit keepLongStdio: true, testResults: "${OUTPUT_DIR}/**/*-junit.xml"
            }
        }
    }
    post {
        success {
            emailext (
                subject: "SUCCESSFUL: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'",
                body: """<p>SUCCESSFUL: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]':</p>
                <p>Check console output at &QUOT;<a href='${env.BUILD_URL}'>${env.JOB_NAME} [${env.BUILD_NUMBER}]</a>&QUOT;</p>""",
                recipientProviders: [[$class: 'DevelopersRecipientProvider']]
            )
        }
        failure {
            emailext (
                subject: "FAILED: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'",
                body: """<p>FAILED: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]':</p>
                <p>Check console output at &QUOT;<a href='${env.BUILD_URL}'>${env.JOB_NAME} [${env.BUILD_NUMBER}]</a>&QUOT;</p>""",
                recipientProviders: [[$class: 'DevelopersRecipientProvider']]
            )
        }
    }
}

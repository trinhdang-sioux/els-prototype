#!groovy

pipeline {
    agent any

    options {
        timestamps()
    }

    environment {
        OUTPUT_DIR = "Builds\\${env.BUILD_NUMBER}"
        OUTPUT_TEST = "${OUTPUT_DIR}\\tests"
        OUTPUT_ARTIFACT = "${OUTPUT_DIR}\\artifacts"

        SFDC_USERNAME = ""
        HUB_ORG = "${env.HUB_ORG_DH}"
        SFDC_HOST = "${env.SFDC_HOST_DH}"
        CONNECTED_APP_CONSUMER_KEY = "${env.CONNECTED_APP_CONSUMER_KEY_DH}"
        CONNECTED_APP_JWT_KEY = credentials("${env.JWT_CRED_ID_DH}")

        sfdx = "C:\\Program Files\\Salesforce CLI\\bin\\sfdx"

        PACKAGE_DIR = "mdapi_output_dir"
        PACKAGE_NAME = "els-protoype"

        GIT_COMMIT = ""
    }

    stages {
        stage('checkout SCM') {
            steps {
                scmVars = checkout scm
                GIT_COMMIT = scmVars.GIT_COMMIT
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
                            SFDC_USERNAME=robj.result.username
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
            steps {
                script {
                    status = bat returnStatus: true, script: "sfdx force:source:convert -d ${PACKAGE_DIR}/ --packagename ${PACKAGE_NAME}"
                    if(status != 0) {
                        error 'package failed'
                    }
                    bat script: "dir /s /b mdapi_output_dir"
                    zip zipFile: "${OUTPUT_ARTIFACT}\\${env.GIT_COMMIT}.${env.BUILD_NUMBER}.zip", archive: false, dir: "${PACKAGE_DIR}"
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
        always {
            echo 'Post always'
        }
        failure {
            echo 'Email'
        }
    }
}

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

        SFDC_URL = "https://login.salesforce.com"
        SFDC_USERNAME = "trinh.dang@sioux.asia"
        SFDC_ALIAS = "CI"

        SFDC_SANDBOX_URL = "https://test.salesforce.com"
        SFDC_SANDBOX_USERNAME = "trinh.dang@sioux.asia.dev"
        SFDC_SANDBOX_ALIAS = "DEV"

        // TODO change this to secrect text also
        CONNECTED_APP_CONSUMER_KEY = "3MVG9YDQS5WtC11qeOgeko3X5nfieoVD3Lg_0DhCjdjB2MPPWhv9JZugQNKrPi1esWVOm6_6Y3zPu3iI0KTbf"
        CONNECTED_APP_JWT_KEY = credentials("SALESFORCE_PRIVATE_KEY")

        sfdx = "C:\\Program Files\\Salesforce CLI\\bin\\sfdx"
    }

    stages {
        stage('create output dir') {
            steps {
                script {
                    deleteDir()
                    bat script: "mkdir ${OUTPUT_TEST}"
                    bat script: "mkdir ${OUTPUT_ARTIFACT}"
                }
            }
        }

        stage('checkout SCM') {
            steps {
                script {
                    scmVars = checkout scm
                    COMMIT_NUMBER = scmVars.GIT_COMMIT
                }
            }
        }

        stage('build and test') {
            stages {
                stage('authorize dev hub org') {
                    steps {
                        script {
                            status = bat returnStatus: true, script: "\"${sfdx}\" force:auth:jwt:grant --clientid ${CONNECTED_APP_CONSUMER_KEY} --username ${SFDC_USERNAME} --jwtkeyfile \"${CONNECTED_APP_JWT_KEY}\" --instanceurl ${SFDC_URL} --setdefaultdevhubusername"
                            if (status != 0) {
                                error 'Org authorization failed'
                            }
                        }
                    }
                }

                stage('create scratch org') {
                    steps {
                        script {
                            stdout = bat returnStatus: true, script: "\"${sfdx}\" force:org:create --definitionfile config/project-scratch-def.json --json --setalias ${SFDC_ALIAS}"
                            if (status != 0) {
                                error 'Org authorization failed'
                            }
                        }
                    }
                }

                stage('push source to scratch org') {
                    steps {
                        script {
                            status = bat returnStatus: true, script: "\"${sfdx}\" force:source:push --targetusername ${SFDC_ALIAS}"
                            if (status != 0) {
                                    error 'Org push failed'
                            }
                            status = bat returnStatus: true, script: "\"${sfdx}\" force:user:permset:assign --targetusername ${SFDC_ALIAS} --permsetname ELS"
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
                                rc = bat returnStatus: true, script: "\"${sfdx}\" force:apex:test:run --testlevel RunLocalTests --outputdir ${OUTPUT_TEST} --resultformat tap --codecoverage --targetusername ${SFDC_ALIAS}"
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
                        status = bat returnStatus: true, script: "\"${sfdx}\" force:org:delete --targetusername ${SFDC_ALIAS} --noprompt"
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
                    status = bat returnStatus: true, script: "\"${sfdx}\" force:source:convert --deploydir  ${PACKAGE_DIR}/ --packagename ${PACKAGE_NAME}"
                    if(status != 0) {
                        error 'package failed'
                    }
                    bat script: "dir /s /b mdapi_output_dir"
                    zip zipFile: "${PACKAGE_ZIP}", archive: false, dir: "${PACKAGE_DIR}"
                }
            }
        }

        stage('deploy to staging') {
            stages {
                stage('authorize sandbox org') {
                    steps {
                        script {
                            status = bat returnStatus: true, script: "\"${sfdx}\" force:auth:jwt:grant --clientid ${CONNECTED_APP_CONSUMER_KEY} --username ${SFDC_SANDBOX_USERNAME} --jwtkeyfile \"${CONNECTED_APP_JWT_KEY}\" -instanceurl ${SFDC_SANDBOX_URL} --setalias ${SFDC_SANDBOX_ALIAS}"
                            if(status != 0) {
                                error 'authorize sandbox failed'
                            }
                        }
                    }
                }
                stage('validate deployment') {
                    steps {
                        script {
                            status = bat returnStatus: true, script: "\"${sfdx}\" force:mdapi:deploy --zipfile ${PACKAGE_ZIP} --testlevel RunAllTestsInOrg --targetusername ${SFDC_SANDBOX_ALIAS} --wait 10 --checkonly"
                            if(status != 0) {
                                error 'validate deployment failed'
                            }
                        }
                    }
                }

                stage('deploy') {
                    steps {
                        script {
                            status = bat returnStatus: true, script: "\"${sfdx}\" force:mdapi:deploy --zipfile ${PACKAGE_ZIP} --targetusername ${SFDC_SANDBOX_ALIAS} --wait 10"
                            if(status != 0) {
                                error 'deploy failed'
                            }
                        }
                    }
                }
            }
            post {
                always {
                    script {
                        bat script: "\"${sfdx}\" force:auth:logout --targetusername ${SFDC_SANDBOX_ALIAS} --noprompt"
                    }
                }
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
                to: "khoa.nguyen@sioux.asia",
                recipientProviders: [[$class: 'DevelopersRecipientProvider']]
            )
        }
        failure {
            emailext (
                subject: "FAILED: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'",
                body: """<p>FAILED: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]':</p>
                <p>Check console output at &QUOT;<a href='${env.BUILD_URL}'>${env.JOB_NAME} [${env.BUILD_NUMBER}]</a>&QUOT;</p>""",
                to: "khoa.nguyen@sioux.asia",
                recipientProviders: [[$class: 'DevelopersRecipientProvider']]
            )
        }
    }
}

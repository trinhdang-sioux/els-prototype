#!groovy

pipeline {
    agent any

    options {
        timestamps()
    }

    environment {
        ARTIFACT_DIR = "Builds\\${env.BUILD_NUMBER}"

        HUB_ORG = env.HUB_ORG_DH
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

        stage('Build and test') {
            stages {
                stage('Authorize DEV HUB org') {
                    steps {
                        echo HUB_ORG
                    }
                }

                stage('Create SCRATCH org') {
                    steps {

                    }
                }

                stage('Push to SCRATCH org') {
                    steps {

                    }
                }

                stage('Run APEX tests') {
                    steps {

                    }
                }
            }
        }
        post {
            failure {

            }
        }

        stage('Deploy') {
            steps {

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
            echo 'cleanup'
        }
        failure {
            // notify users when the Pipeline fails
            mail to: 'trinh.dang@sioux.asia',
            subject: "Failed Pipeline: ${currentBuild.fullDisplayName}",
            body: "Something is wrong with ${env.BUILD_URL}"
        }
    }
}

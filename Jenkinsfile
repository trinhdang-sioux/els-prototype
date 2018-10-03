#!groovy

pipeline {
    agent any

    options {
        timestamps()
    }

    environment {
        ARTIFACT_DIR = "Builds\\${env.BUILD_NUMBER}"

        HUB_ORG = "${env.HUB_ORG_DH}"
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
                        echo ARTIFACT_DIR
                    }
                }

                stage('Create SCRATCH org') {
                    steps {
                        echo 'Create SCRATCH org'
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
                    echo 'Cleanup org here'
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
            
        }
        failure {
            // notify users when the Pipeline fails
            mail to: 'trinh.dang@sioux.asia',
            subject: "Failed Pipeline: ${currentBuild.fullDisplayName}",
            body: "Something is wrong with ${env.BUILD_URL}"
        }
    }
}

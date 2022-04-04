def isSASTEnabled
def isSCAEnabled
def isDASTEnabled
def isSASTPlusMEnabled
def isDASTPlusMEnabled
def isImageScanEnabled
def buildBreakerStatus

pipeline {
    agent any
    tools {
        maven 'Maven3'
    }

    stages {
        stage('Checkout Source Code') {
            steps {
                git branch: 'master', url: 'https://github.com/io-poc/sas-poc-java'
            }
        }

        stage('Build Source Code') {
            steps {
                sh '''mvn clean compile'''
            }
        }

        stage('IO - Prescription') {
            steps {
                synopsysIO(
                    connectors: [
                        io(configName: 'io-poc',
                           projectName: 'sas-poc-java',
                           workflowVersion: '2021.12.4'),
                        github(branch: 'master',
                               configName: 'poc-github',
                               owner: 'io-poc',
                               repositoryName: 'sas-poc-java'),
                        codeDx(configName: 'poc-codedx',
                               projectId: '2'),
                        buildBreaker(configName: 'poc-bb')]) {
                    sh 'io --version'
                    sh 'io --stage io --verbose'
                    }

                script {
                    def prescriptionJSON = readJSON file: 'io_state.json'

                    isSASTEnabled = prescriptionJSON.data.prescription.security.activities.sast.enabled
                    isSASTPlusMEnabled = prescriptionJSON.data.prescription.security.activities.sastPlusM.enabled
                    isSCAEnabled = prescriptionJSON.data.prescription.security.activities.sca.enabled
                    isDASTEnabled = prescriptionJSON.data.prescription.security.activities.dast.enabled
                    isDASTPlusMEnabled = prescriptionJSON.data.prescription.security.activities.dastPlusM.enabled
                    isImageScanEnabled = prescriptionJSON.data.prescription.security.activities.imageScan.enabled
                }
            }
        }

        stage('SAST - Polaris') {
            when {
                expression { isSASTEnabled }
            }
            steps {
                echo 'Polaris - Placeholder - TODO'
            }
        }

        stage('SCA - BlackDuck') {
            when {
                expression { isSCAEnabled }
            }
            steps {
              echo 'Running SCA using BlackDuck'
              synopsysIO(connectors: [
                  blackduck(configName: 'poc-bd',
                  projectName: 'sas-poc-java',
                  projectVersion: '0.0.1-SNAPSHOT')]) {
                  sh 'io --stage execution --state io_state.json'
              }
            }
        }

        stage('IO - Workflow') {
            steps {
                echo 'Execute Workflow Stage'
                synopsysIO(connectors: [
                    msteams(configName: 'io-bot'), ]) {
                    sh 'io --stage workflow --state io_state.json'
                }
            }
        }

        stage('Security Sign-Off') {
            steps {
                script {
                    if (fileExists('wf-output.json')) {
                        def workflowJSON = readJSON file: 'wf-output.json'
                        buildBreakerStatus = workflowJSON.breaker.status

                        print("========================== IO WorkflowEngine Summary ============================")
                        print("Build Breaker Status: $buildBreakerStatus")

                        if(workflowJSON.summary.size() > 0) {
                            workflowJSON.summary.each{ activity->
                                print("Activity: ${activity.activity}")
                                if(activity.has("breakercount")) {
                                    breakerCount = activity.breakercount.size()
                                    print("Build Breaker Count: $breakerCount")
                                    if (breakerCount > 0) {
                                        activity.breakercount.each{ breaker ->
                                            print("Severity: ${breaker.severity}")
                                            print("Count: ${breaker.count}")
                                        }
                                    }
                                } else if(activity.has("risk_score")) {
                                    print("Code Dx Risk Score: ${activity.risk_score}")
                                }
                            }
                        } else {
                            print("No workflow summary available.")
                        }
                        print("========================== IO WorkflowEngine Summary ============================")
                        
                        if (buildBreakerStatus) {
                            input message: 'One or more conditions triggered Build Breaker. Do you wish to proceed?'
                        }
                    } else {
                        print("Workflow Engine output not available.")
                    }
                }
                echo "Security Sign-Off Check Complete"
            }
        }
    }

    // post {
    //     always {
    //         // Always remove the state JSON file as it has sensitive information
    //         sh 'rm io_state.json'
    //     }
    // }
}

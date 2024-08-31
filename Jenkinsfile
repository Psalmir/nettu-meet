pipeline {
    agent any
    environment {
        defectDojoApiKey = 'c5b50032ffd2e0aa02e2ff56ac23f0e350af75b4'
        defectDojoUrl = 'https://s410-exam.cyber-ed.space:8083/api/v2/'
        semgrepReport = 'semgrep-report.json'
        trivyReport = 'trivy-report.json'
        dependencyTrackUrl = 'https://s410-exam.cyber-ed.space:8081/'
        dependencyTrackApiKey = 'odt_SfCq7Csub3peq7Y6lSlQy5Ngp9sSYpJl'
     }
    stages {
        stage('SAST with Semgrep') {
            agent {
                label 'alpine'
            }
            when {
                expression { true }
            }

            steps {
                script {
                    try {
                        sh '''
                            apk update && apk add --no-cache python3 py3-pip py3-virtualenv
                            python3 -m venv venv
                            . venv/bin/activate
                            pip install semgrep
                            semgrep ci --config auto --json > ${semgrepReport}
                        '''
                    } catch (Exception e) {
                        echo 'Semgrep encountered issues.'
                    }
                }

                sh 'ls -lth'
                stash name: 'semgrep-report', includes: "${semgrepReport}"
                archiveArtifacts artifacts: "${semgrepReport}", allowEmptyArchive: true
            }
        } 
        stage('Run ZAP Scan') {
            agent {
                label 'alpine'
            }    

            steps {
                sh 'pwd'
                sh 'curl -L -o ZAP_2.15.0_Linux.tar.gz https://github.com/zaproxy/zaproxy/releases/download/v2.15.0/ZAP_2.15.0_Linux.tar.gz'
                sh 'tar -xzf ZAP_2.15.0_Linux.tar.gz'
                sh './ZAP_2.15.0/zap.sh -cmd -addonupdate -addoninstall wappalyzer -addoninstall pscanrulesBeta'
                sh './ZAP_2.15.0/zap.sh -cmd -quickurl https://s410-exam.cyber-ed.space:8084 -quickout $(pwd)/zapsh-report.json'
                stash name: 'zapsh-report', includes: 'zapsh-report.json'
                archiveArtifacts artifacts: 'zapsh-report.json', allowEmptyArchive: true         
            }            
        }
        stage('Reports') {
            agent {
                label 'alpine'
            }    
            steps {
                sh 'cp ./reports/* ./'
                sh 'ls -lt'
                // stash name: 'sbom', includes: 'sbom.json'
                stash name: 'semgrep-report', includes: "${semgrepReport}"
                stash name: 'zapsh-report', includes: 'zapsh-report.json'
            }            
        }

        // stage('Upload Reports to DefectDojo') {
        //     agent {
        //         label 'alpine'
        //     } 
        //     steps {
        //         script {
        //             sh """
        //                 ls -lth 
        //                 curl -k -X POST -H "Authorization: Token ${defectDojoApiKey}" \
        //                 -F "engagement=null" \
        //                 -F "scan_type=Semgrep" \
        //                 -F "file=@${semgrepReport}" \
        //                 ${defectDojoUrl}import-scan/
        //             """

        //             sh """
        //                 ls -lth 
        //                 curl -k -X POST -H "Authorization: Token ${defectDojoApiKey}" \
        //                 -F "engagement=null" \
        //                 -F "scan_type=ZAP" \
        //                 -F "file=@${zapshReport}" \
        //                 ${defectDojoUrl}import-scan/
        //             """
        //         }
        //     }
        // }      
    }
}

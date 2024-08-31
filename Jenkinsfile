pipeline {
    agent any
    environment {
        defectDojoApiKey = 'c5b50032ffd2e0aa02e2ff56ac23f0e350af75b4'
        defectDojoUrl = 'https://s410-exam.cyber-ed.space:8083/api/v2'
        semgrepReport = 'semgrep-report.json'
        trivyReport = 'trivy-report.json'
        dependencyTrackUrl = 'https://s410-exam.cyber-ed.space:8081/'
        dependencyTrackApiKey = 'odt_SfCq7Csub3peq7Y6lSlQy5Ngp9sSYpJl'
     }
    stages {
        // stage('SAST with Semgrep') {
        //     agent {
        //         label 'alpine'
        //     }
        //     when {
        //         expression { true }
        //     }

        //     steps {
        //         script {
        //             try {
        //                 sh '''
        //                     apk update && apk add --no-cache python3 py3-pip py3-virtualenv
        //                     python3 -m venv venv
        //                     . venv/bin/activate
        //                     pip install semgrep
        //                     semgrep ci --config auto --json > ${semgrepReport}
        //                 '''
        //             } catch (Exception e) {
        //                 echo 'Semgrep encountered issues.'
        //             }
        //         }

        //         sh 'ls -lth'
        //         stash name: 'semgrep-report', includes: "${semgrepReport}"
        //         archiveArtifacts artifacts: "${semgrepReport}", allowEmptyArchive: true
        //     }
        // } 
        // stage('Run ZAP Scan') {
        //     agent {
        //         label 'alpine'
        //     }    

        //     steps {
        //         sh 'pwd'
        //         sh 'curl -L -o ZAP_2.15.0_Linux.tar.gz https://github.com/zaproxy/zaproxy/releases/download/v2.15.0/ZAP_2.15.0_Linux.tar.gz'
        //         sh 'tar -xzf ZAP_2.15.0_Linux.tar.gz'
        //         sh './ZAP_2.15.0/zap.sh -cmd -addonupdate -addoninstall wappalyzer -addoninstall pscanrulesBeta'
        //         sh './ZAP_2.15.0/zap.sh -cmd -quickurl https://s410-exam.cyber-ed.space:8084 -quickout $(pwd)/zapsh-report.json'
        //         stash name: 'zapsh-report', includes: 'zapsh-report.json'
        //         archiveArtifacts artifacts: 'zapsh-report.json', allowEmptyArchive: true         
        //     }            
        // }
        // stage('SCA') {
        //     agent {
        //         label 'trivy'
        //     }
        //     when {
        //         expression { true }
        //     }

        //     steps {
        //         sh '''
        //             cd server
        //             docker build . -t ${DOCKER_IMAGE_NAME} -fDockerfile
        //             #echo '192.168.5.13 harbor.cyber-ed.labs' >> /etc/hosts
        //             #sudo curl -sSfL https://raw.githubusercontent.com/anchore/grype/main/install.sh | sudo sh -s -- -b /usr/local/bin
        //             #curl -sSfL https://raw.githubusercontent.com/anchore/syft/main/install.sh | sh -s -- -b /usr/local/bin
        //             docker image ls
        //             #sudo grype docker:podkatilovas/pygoat:113 -o table >> ${SCA_REPORT}
        //             ls -lt
        //         '''
        //         stash name: 'semgrep-report', includes: "${SCA_REPORT}"
        //         archiveArtifacts artifacts: "${SCA_REPORT}", allowEmptyArchive: true
        //     }
        // } 

        stage('Upload Reports to DefectDojo') {
            agent {
                label 'alpine'
            } 
            steps {
                script {
                    sh """
                        curl -k -X POST -H "Authorization: Token ${defectDojoApiKey}" -H "Content-Type: application/json" \
                        -d '{
                        "engagement": null,
                        "scan_type": "Semgrep",
                        "file": "@${semgrepReport}"
                        }' ${defectDojoUrl}import-scan/
                    """

                    sh """
                        curl -k -X POST -H "Authorization: Token ${defectDojoApiKey}" -H "Content-Type: application/json" \
                        -d '{
                            "engagement": null,
                            "scan_type": "ZAP",
                            "file": "@${zapshReport}"
                        }' ${defectDojoUrl}import-scan/
                    """
                }
            }
        }      
    }
}

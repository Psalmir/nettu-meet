pipeline {
    agent any
    stages {
        stage('SAST with Semgrep') {
            steps {
                script {
                    def results = sh(script: 'semgrep --config auto .', returnStdout: true)
                    if (results.contains('ERROR')) {
                        currentBuild.result = 'FAILURE'
                        error("Semgrep found issues.")
                    }
                    // Отправка результатов в Defect Dojo (пример)
                    // sh "curl -X POST -H 'Authorization: Token ${DEFECT_DOJO_TOKEN}' -d '${results}' ${DEFECT_DOJO_URL}/api/v2/import-scan/"
                }
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
    }
}



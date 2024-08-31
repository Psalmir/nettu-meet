pipeline {
    agent any
    environment {
         semgrepReport = 'semgrep-report.json'
     }
    stages {
        stage('SAST with Semgrep') {
            agent {
                label 'alpine'
            }

            steps {
                script {
                        def semgrepReport = 'semgrep-report.json'
                        sh '''
                            apk update && apk add --no-cache python3 py3-pip py3-virtualenv
                            python3 -m venv venv
                            . venv/bin/activate
                            pip install semgrep
                        '''
                    
                }

                  // Выполнение Semgrep и сохранение отчета
            try {
                sh "semgrep ci --config auto --json > ${semgrepReport}"
            } catch (Exception e) {
                echo 'An error occurred while running Semgrep.'
                currentBuild.result = 'FAILURE'
                return
            }

            // Перечисление файлов для проверки
            sh 'ls -lth'

            // Сохранение отчета
            stash name: 'semgrep-report', includes: "${semgrepReport}"
            archiveArtifacts artifacts: "${semgrepReport}", allowEmptyArchive: true
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




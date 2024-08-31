pipeline {
    agent any
    stages {
        stage('SAST with Semgrep') {
    agent {
        label 'alpine'
    }

    steps {
        script {
            def semgrepReport = 'semgrep-report.json'
            sh '''
                apk update
                apk add --no-cache python3 py3-pip
                python3 -m pip install --no-cache-dir semgrep
            '''
            try {
                sh "semgrep ci --config auto --json > ${semgrepReport}"
            } catch (Exception e) {
                echo 'An error occurred while running Semgrep.'
                currentBuild.result = 'FAILURE'
                return
            }
            sh 'ls -lth'
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
}



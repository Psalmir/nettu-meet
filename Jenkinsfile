pipeline {
    agent any
    environment {
        defectDojoApiKey = 'c5b50032ffd2e0aa02e2ff56ac23f0e350af75b4'
        defectDojoUrl = 'https://s410-exam.cyber-ed.space:8083/api/v2/'
        semgrepReport = 'semgrep-report.json'
        trivyReportDir = 'reports/'
        dependencyTrackUrl = 'https://s410-exam.cyber-ed.space:8081/'
        dependencyTrackApiKey = 'odt_SfCq7Csub3peq7Y6lSlQy5Ngp9sSYpJl'
        projectName = 'psalmir'
        projectVersion = '1.0'
        projectDescription = 'Exam'
     }
    stages {
        stage('Semgrep') {
            agent { label 'alpine' }
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
                archiveArtifacts artifacts: "${semgrepReport}", allowEmptyArchive: true
                stash name: 'semgrep-report', includes: "${semgrepReport}"
            }
        } 
        stage('ZAP') {
            agent { label 'alpine' }    
            steps {
                sh '''
                    curl -L -o ZAP_2.15.0_Linux.tar.gz https://github.com/zaproxy/zaproxy/releases/download/v2.15.0/ZAP_2.15.0_Linux.tar.gz
                    tar -xzf ZAP_2.15.0_Linux.tar.gz
                    ./ZAP_2.15.0/zap.sh -cmd -addonupdate -addoninstall wappalyzer -addoninstall pscanrulesBeta
                    ./ZAP_2.15.0/zap.sh -cmd -quickurl https://s410-exam.cyber-ed.space:8084 -quickout $(pwd)/zapsh-report.json
                '''
                archiveArtifacts artifacts: 'zapsh-report.json', allowEmptyArchive: true 
                stash name: 'zapsh-report', includes: 'zapsh-report.json'
            }            
        }
        stage ('SBOM') {
          steps {
            script {
                sh '''
                    curl -sSfL https://raw.githubusercontent.com/anchore/syft/main/install.sh | sh -s -- -b ./bin
                    export PATH=$(pwd)/bin:$PATH
                    syft dir:$(pwd) -o cyclonedx-xml=bom.xml
                    curl -k -X "PUT" "${dependencyTrackUrl}/api/v1/project" \
                         -H "X-API-Key: ${dependencyTrackApiKey}" \
                         -H "Content-Type: application/json" \
                         -d '{
                              "name": "${projectName}",
                              "version": "${projectVersion}",
                              "description": "${projectDescription}"
                             }'
                    curl -k -X "POST" "${dependencyTrackUrl}/api/v1/bom" \
                         -H "Content-Type: multipart/form-data" \
                         -H "X-Api-Key: ${dependencyTrackApiKey}" \
                         -F "autoCreate=true" \
                         -F "projectName=${projectName}" \
                         -F "projectVersion=${projectVersion}" \
                         -F "bom=@bom.xml"
                '''
            }
        }
      }
      stage('Trivy') {
            agent { label 'dind' }
            steps {
                script {
                    sh '''
                        docker pull aquasec/trivy:0.54.1
                        docker run --rm -v $(pwd):/project aquasec/trivy:0.54.1 repo --format json --output /project/trivy_report.json /project
                    '''
                }
                archiveArtifacts artifacts: 'trivy_report.json', allowEmptyArchive: true
            }
        }
        // stage('Quality Gate for SAST') {
        //     steps {
        //         script {
        //             def highCount = sh(script: "grep -o '\"impact\": \"HIGH\"' ${semgrepReport} | wc -l", returnStdout: true).trim().toInteger()
        //             def criticalCount = sh(script: "grep -o '\"impact\": \"CRITICAL\"' ${semgrepReport} | wc -l", returnStdout: true).trim().toInteger()
                    
        //             if (highCount > 5) {
        //                 error "Semgrep: High severity issues found! (${highCount} high severity issues)"
        //             }
                    
        //             if (criticalCount > 1) {
        //                 error "Semgrep: Critical severity issues found! (${criticalCount} critical severity issues)"
        //             }
        //         }
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

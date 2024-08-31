pipeline {
    agent { 
        label 'alpine'
    }
    environment {
        DEFECT_DOJO_KEY = 'c5b50032ffd2e0aa02e2ff56ac23f0e350af75b4'
        DEFECT_DOJO_URL = 'https://s410-exam.cyber-ed.space:8083'
        DEPENDENCY_TRACK_URL = 'https://s410-exam.cyber-ed.space:8081'
        DEPENDENCY_TRACK_API_KEY = 'odt_SfCq7Csub3peq7Y6lSlQy5Ngp9sSYpJl'
        PROJECT_NAME = 'psalmir'
        PROJECT_VERSION = '1.0'
        PROJECT_DESCRIPTION = 'Exam'
        PROJECT_UUID = '24e2a077-7195-4d91-8cd0-2552c0330aac'
        STAND_URL = 'https://s410-exam.cyber-ed.space:8084'
     }
    stages {
        stage('Semgrep') {
            agent { label 'alpine' }
            steps {
                script {
                        sh '''
                            apk update && apk add --no-cache python3 py3-pip py3-virtualenv
                            python3 -m venv venv
                            . venv/bin/activate
                            pip install semgrep
                            . venv/bin/activate && semgrep --config=auto . --json > semgrep_report.json
                        '''
                        archiveArtifacts artifacts: 'semgrep_report.json', allowEmptyArchive: true
                        stash name: 'reports/semgrep_report.json', includes: "semgrep-report"
                }
            }
        } 
        stage('ZAP') {
            agent { label "dind" }
        steps {
          sh '''
          mkdir reports/
          sudo apt update
          sudo apt install -y wget openjdk-11-jre
          wget -q https://github.com/zaproxy/zaproxy/releases/download/v2.15.0/ZAP_2.15.0_Linux.tar.gz
          tar xzf ZAP_2.15.0_Linux.tar.gz        
          ZAP_2.15.0/zap.sh -cmd -addonupdate -addoninstall wappalyzer -addoninstall pscanrulesBeta
          ZAP_2.15.0/zap.sh -cmd -quickurl https://s410-exam.cyber-ed.space:8084 -quickout $(pwd)/zapout.json
          cp $(pwd)/zapout.json reports/
          '''
          archiveArtifacts artifacts: 'reports/*', allowEmptyArchive: true
          stash includes: 'reports/zapout.json', name: 'zapout'
        }
      }

        stage('Dependency Track') {
            steps {
                script{
                    sh '''
                    curl -sSfL https://raw.githubusercontent.com/anchore/syft/main/install.sh | sh -s -- -b /usr/local/bin
                    syft dir:$(pwd) -o cyclonedx-xml@1.5 > sbom.xml

                    curl -k -X "PUT" "https://s410-exam.cyber-ed.space:8081/api/v1/bom" \
                    -H 'Content-Type: application/json' \
                    -H 'X-API-Key: odt_SfCq7Csub3peq7Y6lSlQy5Ngp9sSYpJl' \
                    -d '{
                        "project": "e24b8a18-0695-4ec0-b7fe-25e6e14b22d6",
                        "bom": {
                            "format": "CycloneDX",
                            "data": "'$(sbom.json)'"
                                }
                        }'
                    '''
                    archiveArtifacts artifacts: 'sbom.json', allowEmptyArchive: true
                }
            }
        }
        stage ('Trivy') {
          agent { label "dind" }
          steps {
            script {
                sh '''
                wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo apt-key add -
                echo deb https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main | sudo tee -a /etc/apt/sources.list.d/trivy.list
                sudo apt update
                sudo apt install trivy
                mkdir reports/
                cd server
                trivy fs --format cyclonedx -o ../reports/sbom.json package-lock.json
                #docker build . -t nettu-meet:latest -f Dockerfile
                #trivy image --format cyclonedx -o ../reports/sbom.json nettu-meet-server:latest
                trivy sbom -f json -o ../reports/trivy.json ../reports/sbom.json
                '''
                archiveArtifacts artifacts: 'reports/*', allowEmptyArchive: true
                stash includes: 'reports/sbom.json', name: 'sbom'
                stash includes: 'reports/trivy.json', name: 'trivy-report'
              }
          }
      }
            // stage('Defect Dojo') {
            //     steps {
            //         script {
            //             // Установка необходимых утилит
            //             sh 'apk add --no-cache curl jq'

            //             // Создание тестового набора в Defect Dojo
            //             def createTestSuiteResponse = sh(
            //                 script: """
            //                 curl -k -X "POST" "${DEFECT_DOJO_URL}/api/v2/test_suites/" \
            //                      -H "Authorization: Token ${DEFECT_DOJO_KEY}" \
            //                      -H "Content-Type: application/json" \
            //                      -d '{
            //                           "name": "Security Test Suite",
            //                           "description": "Automated security scan results"
            //                          }'
            //                 """,
            //                 returnStdout: true
            //             ).trim()

            //             def testSuiteId = sh(script: "echo '${createTestSuiteResponse}' | jq -r '.id'", returnStdout: true).trim()

            //             if (!testSuiteId) {
            //                 error "Failed to create test suite in Defect Dojo"
            //             }

            //             echo "Test Suite ID: ${testSuiteId}"

            //             // Функция для загрузки артефактов в Defect Dojo
            //             def uploadToDefectDojo = { file, scanType ->
            //                 sh """
            //                 curl -k -X "POST" "${DEFECT_DOJO_URL}/api/v2/import_scan/" \
            //                      -H "Authorization: Token ${DEFECT_DOJO_KEY}" \
            //                      -H "Content-Type: multipart/form-data" \
            //                      -F "file=@${file}" \
            //                      -F "test_type=${scanType}" \
            //                      -F "test_suite=${testSuiteId}" \
            //                      -F "scan_type=${scanType}"
            //                 """
            //             }

            //             // Пример использования функции для загрузки файлов
            //             uploadToDefectDojo('semgrep_report.json', 'Semgrep')
            //             uploadToDefectDojo('zapout.json', 'ZAP')
            //             uploadToDefectDojo('sbom.json', 'Dependency Track')
            //             // uploadToDefectDojo('trivy-repo-report.json', 'Trivy')
            //         }
            //     }
            // }




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
    }
}

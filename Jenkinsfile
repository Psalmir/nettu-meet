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
                }
            }
        } 
        stage('ZAP') {
            agent { label 'alpine' }    
            steps{
                script{
                    sh '''
                    docker run -v \$(pwd)/:/zap/wrk/:rw -t zaproxy/zap-stable zap-baseline.py -I -t https://s410-exam.cyber-ed.space:8084 -J zapout.json  
                    '''
                    archiveArtifacts artifacts: 'zapout.json', allowEmptyArchive: true
                }
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
        // stage('Trivy') {
        //     agent {
        //         label 'dind'
        //     }
        //     steps {
        //         script {
        //             sh '''
                    
        //             docker pull aquasec/trivy:0.54.1

                    
        //             docker run --rm -v $(pwd):/project aquasec/trivy:0.54.1 repo --format json --output /project/trivy-repo-report.json /project
        //             '''
        //         }
        //     }
        //     post {
        //         always {
        //             archiveArtifacts artifacts: 'trivy-repo-report.json', allowEmptyArchive: true, caseSensitive: false, defaultExcludes: false, followSymlinks: false
        //         }
        //     }
        // }
            stage('Defect Dojo') {
                steps {
                    script {
                        // Установка необходимых утилит
                        sh 'apk add --no-cache curl jq'

                        // Создание тестового набора в Defect Dojo
                        def createTestSuiteResponse = sh(
                            script: """
                            curl -k -X "POST" "${DEFECT_DOJO_URL}/api/v2/test_suites/" \
                                 -H "Authorization: Token ${DEFECT_DOJO_KEY}" \
                                 -H "Content-Type: application/json" \
                                 -d '{
                                      "name": "Security Test Suite",
                                      "description": "Automated security scan results"
                                     }'
                            """,
                            returnStdout: true
                        ).trim()

                        def testSuiteId = sh(script: "echo '${createTestSuiteResponse}' | jq -r '.id'", returnStdout: true).trim()

                        if (!testSuiteId) {
                            error "Failed to create test suite in Defect Dojo"
                        }

                        echo "Test Suite ID: ${testSuiteId}"

                        // Функция для загрузки артефактов в Defect Dojo
                        def uploadToDefectDojo = { file, scanType ->
                            sh """
                            curl -k -X "POST" "${DEFECT_DOJO_URL}/api/v2/import_scan/" \
                                 -H "Authorization: Token ${DEFECT_DOJO_KEY}" \
                                 -H "Content-Type: multipart/form-data" \
                                 -F "file=@${file}" \
                                 -F "test_type=${scanType}" \
                                 -F "test_suite=${testSuiteId}" \
                                 -F "scan_type=${scanType}"
                            """
                        }

                        // Пример использования функции для загрузки файлов
                        uploadToDefectDojo('semgrep_report.json', 'Semgrep')
                        uploadToDefectDojo('zapout.json', 'ZAP')
                        uploadToDefectDojo('sbom.json', 'Dependency Track')
                        // uploadToDefectDojo('trivy-repo-report.json', 'Trivy')
                    }
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
    }
}

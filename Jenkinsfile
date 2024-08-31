pipeline {
    agent any
    environment {
        DEFECT_DOJO_KEY = 'c5b50032ffd2e0aa02e2ff56ac23f0e350af75b4'
        DEFECT_DOJO_URL = 'https://s410-exam.cyber-ed.space:8083'
        semgrepReport = 'semgrep-report.json'
        DEPENDENCY_TRACK_URL = 'https://s410-exam.cyber-ed.space:8081'
        DEPENDENCY_TRACK_API_KEY = 'odt_SfCq7Csub3peq7Y6lSlQy5Ngp9sSYpJl'
        PROJECT_NAME = 'psalmir'
        PROJECT_VERSION = '1.0'
        PROJECT_DESCRIPTION = 'Exam'
        PROJECT_UUID = '24e2a077-7195-4d91-8cd0-2552c0330aac'
     }
    stages {
        // stage('Semgrep') {
        //     agent { label 'alpine' }
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
        //         archiveArtifacts artifacts: "${semgrepReport}", allowEmptyArchive: true
        //         stash name: 'semgrep-report', includes: "${semgrepReport}"
        //     }
        // } 
        // stage('ZAP') {
        //     agent { label 'alpine' }    
        //     steps {
        //         sh '''
        //             curl -L -o ZAP_2.15.0_Linux.tar.gz https://github.com/zaproxy/zaproxy/releases/download/v2.15.0/ZAP_2.15.0_Linux.tar.gz
        //             tar -xzf ZAP_2.15.0_Linux.tar.gz
        //             ./ZAP_2.15.0/zap.sh -cmd -addonupdate -addoninstall wappalyzer -addoninstall pscanrulesBeta
        //             ./ZAP_2.15.0/zap.sh -cmd -quickurl https://s410-exam.cyber-ed.space:8084 -quickout $(pwd)/zapsh-report.json
        //         '''
        //         archiveArtifacts artifacts: 'zapsh-report.json', allowEmptyArchive: true 
        //         stash name: 'zapsh-report', includes: 'zapsh-report.json'
        //     }            
        // }
        stage('SBOM') {
            steps {
                script {
                    sh '''
                    curl -sSfL https://raw.githubusercontent.com/anchore/syft/main/install.sh | sh -s -- -b ./bin
                    export PATH=$(pwd)/bin:$PATH
                    syft dir:$(pwd) -o cyclonedx-xml=bom.xml
                    cat bom.xml
                    '''
                    def createProjectResponse = sh(
                        script: """
                        curl -k -X "PUT" "${DEPENDENCY_TRACK_URL}/api/v1/project" \
                             -H "X-API-Key: ${DEPENDENCY_TRACK_API_KEY}" \
                             -H "Content-Type: application/json" \
                             -d '{
                                  "name": "${PROJECT_NAME}",
                                  "version": "${PROJECT_VERSION}",
                                  "description": "${PROJECT_DESCRIPTION}"
                                 }'
                        """,
                        returnStdout: true
                    ).trim()
                    echo "Create Project Response: ${createProjectResponse}"
        
                    def uploadSbomResponse = sh(
                        script: """
                        curl -k -X "POST" "${DEPENDENCY_TRACK_URL}/api/v1/bom" \
                             -H "Content-Type: multipart/form-data" \
                             -H "X-Api-Key: ${DEPENDENCY_TRACK_API_KEY}" \
                             -F "autoCreate=true" \
                             -F "projectName=${PROJECT_NAME}" \
                             -F "projectVersion=${PROJECT_VERSION}" \
                             -F "bom=@bom.xml"
                        """,
                        returnStdout: true
                    ).trim()
                    echo "Upload SBOM Response: ${uploadSbomResponse}"
                }
            }
        }
        stage('Dependency Track') {
            steps {
                script {
                    // Установка jq
                    sh 'apk add --no-cache jq'
        
                    // Получение UUID проекта
                    def projectUUID = sh(
                        script: """
                        curl -k -X "GET" "${DEPENDENCY_TRACK_URL}/api/v1/project?name=${PROJECT_NAME}&version=${PROJECT_VERSION}" \
                             -H "X-API-Key: ${DEPENDENCY_TRACK_API_KEY}" \
                             -H "Content-Type: application/json" \
                             | jq -r '.[0].uuid'
                        """,
                        returnStdout: true
                    ).trim()
                    
                    if (!projectUUID) {
                        error "Failed to get project UUID"
                    }
        
                    // Получение уязвимостей и сохранение в файл
                    sh """
                    curl -k -X "GET" "${DEPENDENCY_TRACK_URL}/api/v1/finding/project/${projectUUID}" \
                         -H "X-API-Key: ${DEPENDENCY_TRACK_API_KEY}" \
                         -H "Content-Type: application/json" \
                         -o vulnerabilities.json
                    """
                    
                    echo "Vulnerabilities saved to vulnerabilities.json"
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: 'vulnerabilities.json', allowEmptyArchive: true
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
            uploadToDefectDojo('semgrep_report.json', 'Semgrep Scan')
            uploadToDefectDojo('zapout.json', 'OWASP ZAP Scan')
            uploadToDefectDojo('vulnerabilities.json', 'Dependency Track Scan')
            // uploadToDefectDojo('trivy-repo-report.json', 'Trivy Scan')
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

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
                
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: 'semgrep_report.json', allowEmptyArchive: true
                }
            }
        } 
        stage('ZAP') {
            agent { label 'alpine' }    
            steps {
                script {
                    sh '''
                    apk add --no-cache openjdk11-jre-headless wget unzip
                    wget https://github.com/zaproxy/zaproxy/releases/download/w2024-08-27/ZAP_WEEKLY_D-2024-08-27.zip
                    unzip ZAP_WEEKLY_D-2024-08-27.zip -d zap
                    '''
                    sh '''
                    zap/ZAP_D-2024-08-27/zap.sh -cmd -quickurl ${STAND_URL} -quickout /home/jenkins/workspace/exam_timofeev_mv/zapout.json
                    ls -l
                    find . -name "*.json"
                    '''
                    archiveArtifacts artifacts: 'zapout.json', allowEmptyArchive: true, caseSensitive: false, defaultExcludes: false, followSymlinks: false
                }
            }
        }
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
    }
}

pipeline {
    agent any
    environment {
        DEPTRACK_URL = "https://s410-exam.cyber-ed.space:8081"
        DEPTRACK_API_KEY = "odt_SfCq7Csub3peq7Y6lSlQy5Ngp9sSYpJl"
        DOJO_URL = "https://s410-exam.cyber-ed.space:8083"
        DOJO_API_TOKEN = "c5b50032ffd2e0aa02e2ff56ac23f0e350af75b4"
    }
    stages {
        // stage('Semgrep Scan') {
        //     steps {
        //         script {
        //             catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
        //                 sh '''
        //                 echo "Executing Semgrep..."
        //                 apk add --no-cache python3 py3-pip py3-virtualenv
        //                 python3 -m venv venv
        //                 . venv/bin/activate
        //                 pip install semgrep
        //                 mkdir -p reports
        //                 semgrep scan --config=auto . --exclude=venv --json > reports/semgrep.json
        //                 '''
        //                 archiveArtifacts artifacts: 'reports/*', allowEmptyArchive: true
        //                 stash name: 'semgrep-report', includes: 'reports/semgrep.json'
        //             }
        //         }
        //     }
        // }
        stage('Scan with Trivy') {
            agent { label "dind" }
            steps {
                script {
                    sh '''
                        curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh -s -- -b /usr/local/bin v0.54.1
                        trivy --version
                        touch trivy_report.json
                        trivy repo https://github.com/ElladaNuralieva/nettu-meet -f json -o trivy_report.json
                        curl --insecure -X "POST" "https://s410-exam.cyber-ed.space:8081/api/v1/bom" \
                         -H 'Content-Type: multipart/form-data' \
                         -H 'X-Api-Key: odt_SfCq7Csub3peq7Y6lSlQy5Ngp9sSYpJl' \
                         -F "project=e3ea797e-6dcb-4be3-9927-0e3320363109" \
                         -F "bom=@trivy_report.json"
                    '''
                    archiveArtifacts artifacts: 'trivy_report.json', allowEmptyArchive: true
                    // stash includes: 'reports/sbom.json', name: 'sbom'
                    // stash includes: 'reports/trivy.json', name: 'trivy-report'
                }
            }
        }
        // stage('Upload SBOM to Dependency Track') {
        //     agent { label "dind" }
        //     steps {
        //         unstash 'sbom'
        //         script {
        //             sh '''
        //             echo "Uploading SBOM to Dependency Track..."
        //             ls -l reports/sbom.json  # Проверка наличия файла
        //             response_code=$(curl -v -k --silent --output /dev/null --write-out "%{http_code}" \
        //             -X POST "${DEPTRACK_URL}/api/v1/bom" \
        //             -H "Content-Type: multipart/form-data" \
        //             -H "X-Api-Key: ${DEPTRACK_API_KEY}" \
        //             -F "autoCreate=true" \
        //             -F "projectName=psalmir" \
        //             -F "projectVersion=1.0" \
        //             -F "bom=@reports/sbom.json")
        //             echo "Response Code: $response_code"
        //             '''
        //         }
        //     }
        // }
        // stage('OWASP ZAP Scan') {
        //     agent { label "dind" }
        //     steps {
        //         script {
        //             sh '''
        //             echo "Setting up OWASP ZAP..."
        //             mkdir -p reports
        //             sudo apt update
        //             sudo apt install -y wget openjdk-11-jre
        //             wget -q https://github.com/zaproxy/zaproxy/releases/download/v2.15.0/ZAP_2.15.0_Linux.tar.gz
        //             tar xzf ZAP_2.15.0_Linux.tar.gz
        //             ZAP_2.15.0/zap.sh -cmd -addonupdate -addoninstall wappalyzer -addoninstall pscanrulesBeta
        //             ZAP_2.15.0/zap.sh -cmd -quickurl https://s410-exam.cyber-ed.space:8084 -quickout $(pwd)/reports/owaspzap.json
        //             '''
        //             archiveArtifacts artifacts: 'reports/*', allowEmptyArchive: true
        //             stash includes: 'reports/owaspzap.json', name: 'owaspzap-report'
        //         }
        //     }
        // }
        stage('Import Scans to Defect Dojo') {
            steps {
                    script {
                       sh '''
                        curl -X 'POST' -kL 'https://s410-exam.cyber-ed.space:8083/api/v2/import-scan/' \
                        -H 'accept: application/json' \
                        -H 'Authorization: Token c5b50032ffd2e0aa02e2ff56ac23f0e350af75b4' \
                        -H 'Content-Type: multipart/form-data' \
                        -F 'active=true' \
                        -F 'verified=true' \
                        -F 'scan_type=Semgrep Scan' \
                        -F 'product_name=psalmir' \
                        -F 'file=@trivy_report.json' \
                        -F 'engagement=50'                       
                        '''
                    }
                }
            }
        stage('Quality Gates') {
            agent {
                label 'alpine'
            }
            steps {
                unstash 'semgrep-report'
                unstash 'owaspzap-report'

                script {
                    def xmlFileContent = readFile 'reports/owaspzap.json' 
                    def searchString = "<riskcode>3</riskcode>"
                    def lines = xmlFileContent.split('\n')
                    int zapErrorCount = lines.count { line -> line.contains(searchString) }

                    echo "ZAP total error with risk 3 (High): ${zapErrorCount}"

                    if (zapErrorCount > env.SEMGREP_REPORT_MAX_ERROR.toInteger()) {
                        echo "ZAP QG failed."
                        // Для отладки не блочим
                        // error("ZAP QG failed.")
                    }

                    def jsonText = readFile 'reports/semgrep.json' 
                    def json = new groovy.json.JsonSlurper().parseText(jsonText)
                    int errorCount = 0
                    json.results.each { r ->
                        if (r.extra.severity == "ERROR") {
                            errorCount += 1;
                        }
                    }
                    echo "SEMGREP error count: ${errorCount}"
                    if (errorCount > env.SEMGREP_REPORT_MAX_ERROR.toInteger()) {
                        echo "SEMGREP QG failed."
                        // Для отладки не блочим
                        // error("SEMGREP QG failed.")
                    }
                }
            }
        }
    }
}

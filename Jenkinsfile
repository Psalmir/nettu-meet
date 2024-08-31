pipeline {
    agent any
    environment {
        DEPTRACK_URL = "https://s410-exam.cyber-ed.space:8081"
        DEPTRACK_API_KEY = "odt_SfCq7Csub3peq7Y6lSlQy5Ngp9sSYpJl"
        DOJO_URL = "https://s410-exam.cyber-ed.space:8083"
        DOJO_API_TOKEN = "c5b50032ffd2e0aa02e2ff56ac23f0e350af75b4"
    }
    stages {
        stage('Run Semgrep') {
            steps {
                script {
                    catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                        sh '''
                        echo "Executing Semgrep..."
                        apk add --no-cache python3 py3-pip py3-virtualenv
                        python3 -m venv venv
                        . venv/bin/activate
                        pip install semgrep
                        mkdir -p reports
                        semgrep scan --config=auto . --exclude=venv --json > reports/semgrep.json
                        '''
                        archiveArtifacts artifacts: 'reports/*', allowEmptyArchive: true
                        stash name: 'semgrep-report', includes: 'reports/semgrep.json'
                    }
                }
            }
        }
        stage('Scan with Trivy') {
            agent { label "dind" }
            steps {
                script {
                    sh '''
                    wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo apt-key add -
                    echo "deb https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main" | sudo tee /etc/apt/sources.list.d/trivy.list
                    sudo apt update
                    sudo apt install -y trivy
                    mkdir -p reports
                    cd server
                    trivy fs --format cyclonedx -o ../reports/sbom.json package-lock.json
                    trivy sbom -f json -o ../reports/trivy.json ../reports/sbom.json
                    '''
                    archiveArtifacts artifacts: 'reports/*', allowEmptyArchive: true
                    stash includes: 'reports/sbom.json', name: 'sbom'
                    stash includes: 'reports/trivy.json', name: 'trivy-report'
                }
            }
        }
        stage('Upload SBOM to Dependency Track') {
            agent { label "dind" }
            steps {
                unstash 'sbom'
                script {
                    sh '''
                    echo "Uploading SBOM to Dependency Track..."
                    response_code=$(curl -k --silent --output /dev/null --write-out "%{http_code}" \
                    -X POST "${DEPTRACK_URL}/api/v1/bom" \
                    -H "Content-Type: multipart/form-data" \
                    -H "X-Api-Key: ${DEPTRACK_API_KEY}" \
                    -F "autoCreate=true" \
                    -F "projectName=dronloko" \
                    -F "projectVersion=1.0" \
                    -F "bom=@sbom.json")
                    echo "Response Code: $response_code"
                    '''
                }
            }
        }
        stage('OWASP ZAP Scan') {
            agent { label "dind" }
            steps {
                script {
                    sh '''
                    echo "Setting up OWASP ZAP..."
                    mkdir -p reports
                    sudo apt update
                    sudo apt install -y wget openjdk-11-jre
                    wget -q https://github.com/zaproxy/zaproxy/releases/download/v2.15.0/ZAP_2.15.0_Linux.tar.gz
                    tar xzf ZAP_2.15.0_Linux.tar.gz
                    ZAP_2.15.0/zap.sh -cmd -addonupdate -addoninstall wappalyzer -addoninstall pscanrulesBeta
                    ZAP_2.15.0/zap.sh -cmd -quickurl https://s410-exam.cyber-ed.space:8084 -quickout $(pwd)/reports/owaspzap.json
                    '''
                    archiveArtifacts artifacts: 'reports/*', allowEmptyArchive: true
                    stash includes: 'reports/owaspzap.json', name: 'owaspzap-report'
                }
            }
        }
        // stage('Import Scans to Defect Dojo') {
        //     steps {
        //         unstash 'trivy-report'
        //         script {
        //             sh '''
        //             echo "Importing scans to Defect Dojo..."
        //             curl -k -X POST "${DOJO_URL}/api/v2/import-scan/" \
        //             -H "Authorization: Token ${DOJO_API_TOKEN}" \
        //             -H "Content-Type: multipart/form-data" \
        //             -F "scan_type=Trivy Scan" \
        //             -F "file=@reports/trivy.json;type=application/json" \
        //             -F "engagement=2" \
        //             -F "product_name=dronloko"
        //             '''
        //         }
        //     }
        // }
    }
}

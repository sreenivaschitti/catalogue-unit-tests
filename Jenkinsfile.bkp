pipeline {
    agent {
        node {
            label 'roboshop'
        }
    }

    environment {
        appVersion = ""
        ACC_ID     = "717188664763"
        region     = "us-east-1"
    }

    options {
        timeout(time: 10, unit: 'MINUTES')
    }

    stages {

        stage('Read Version') {
            steps {
                script {
                    def packageJson = readJSON file: 'package.json'
                    appVersion = packageJson.version
                    echo "Building version: ${appVersion}"
                }
            }
        }
        stage('install dependencies'){
            steps {
                script {
                    sh """
                        npm install
                    """
                }
            }
        }
        stage('Unit tests'){
            steps {
                script {
                    sh """
                        npm test
                    """
                }
            }
        }
        // 🔷 SONARQUBE ANALYSIS
        // stage('SonarQube Analysis') {
        //     steps {
        //         script {
        //             def scannerHome = tool name: 'sonar-8'

        //             withSonarQubeEnv('sonar-server') {
        //                 sh "${scannerHome}/bin/sonar-scanner"
        //             }
        //         }
        //     }
        // }

        // // 🔷 QUALITY GATE
        // stage('Quality Gate') {
        //     steps {
        //         timeout(time: 1, unit: 'HOURS') {
        //             waitForQualityGate abortPipeline: true
        //         }
        //     }
        // }

        // 🔷 DEPENDABOT
        stage('Dependabot Alerts Check') {
            steps {
                withCredentials([string(credentialsId: 'github-token', variable: 'GITHUB_TOKEN')]) {
                    script {
                        def owner = 'chellojuramu'
                        def repo  = 'catalogue-unit-tests'

                        def response = sh(
                            script: """
                                curl -s -w "\\n%{http_code}" \\
                                    -H "Authorization: Bearer ${GITHUB_TOKEN}" \\
                                    -H "Accept: application/vnd.github+json" \\
                                    -H "X-GitHub-Api-Version: 2022-11-28" \\
                                    "https://api.github.com/repos/${owner}/${repo}/dependabot/alerts?severity=high,critical&state=open&per_page=100"
                            """,
                            returnStdout: true
                        ).trim()

                        def parts      = response.tokenize('\n')
                        def httpStatus = parts[-1].trim()
                        def body       = parts[0..-2].join('\n')

                        if (httpStatus != '200') {
                            error "GitHub API call failed with HTTP ${httpStatus}. Check token permissions (security_events scope required).\nResponse: ${body}"
                        }

                        def alerts = readJSON text: body

                        if (alerts.size() == 0) {
                            echo "✅ No HIGH or CRITICAL Dependabot alerts found. Pipeline continues."
                        } else {
                            echo "🚨 Found ${alerts.size()} HIGH/CRITICAL Dependabot alert(s):"
                            alerts.each { alert ->
                                def pkg      = alert.security_vulnerability?.package?.name ?: 'unknown'
                                def severity = alert.security_advisory?.severity?.toUpperCase() ?: 'UNKNOWN'
                                def summary  = alert.security_advisory?.summary ?: 'No summary'
                                def fixedIn  = alert.security_vulnerability?.first_patched_version?.identifier ?: 'No fix available'
                                echo "  ❌ [${severity}] ${pkg} — ${summary} (Fixed in: ${fixedIn})"
                            }
                            error "Pipeline failed: ${alerts.size()} HIGH/CRITICAL Dependabot alert(s) detected."
                        }
                    }
                }
            }
        }

        // 🔷 BUILD IMAGE
        stage('Build Docker Image') {
            steps {
                script {
                    withAWS(credentials: 'aws-creds', region: "${region}") {
                        sh """
                            docker build -t ${ACC_ID}.dkr.ecr.${region}.amazonaws.com/roboshop/catalogue:${appVersion} .
                        """
                    }
                }
            }
        }

        // 🔷 TRIVY OS SCAN
        stage('Trivy OS Scan') {
            steps {
                script {

                    sh """
                        trivy image \
                        --scanners vuln \
                        --pkg-types os \
                        --severity MEDIUM \
                        --format table \
                        --output trivy-os-report.txt \
                        --exit-code 0 \
                        ${ACC_ID}.dkr.ecr.${region}.amazonaws.com/roboshop/catalogue:${appVersion}
                    """

                    sh 'cat trivy-os-report.txt'

                    def scanResult = sh(
                        script: """
                            trivy image \
                            --scanners vuln \
                            --pkg-types os \
                            --severity HIGH,MEDIUM \
                            --exit-code 1 \
                            ${ACC_ID}.dkr.ecr.${region}.amazonaws.com/roboshop/catalogue:${appVersion}
                        """,
                        returnStatus: true
                    )

                    if (scanResult != 0) {
                        error "Trivy OS vulnerabilities found!"
                    }
                }
            }
        }

        // 🔷 TRIVY DOCKERFILE SCAN
        // stage('Trivy Dockerfile Scan') {
        //     steps {
        //         script {

        //             sh """
        //                 trivy config \
        //                 --severity HIGH,MEDIUM \
        //                 --format table \
        //                 --output trivy-dockerfile-report.txt \
        //                 Dockerfile
        //             """

        //             sh 'cat trivy-dockerfile-report.txt'

        //             def scanResult = sh(
        //                 script: """
        //                     trivy config \
        //                     --severity HIGH,MEDIUM \
        //                     --exit-code 1 \
        //                     Dockerfile
        //                 """,
        //                 returnStatus: true
        //             )

        //             if (scanResult != 0) {
        //                 error "Dockerfile misconfiguration found!"
        //             }
        //         }
        //     }
        // }

        // 🔷 PUSH TO ECR
        stage('Push to ECR') {
            steps {
                script {
                    withAWS(credentials: 'aws-creds', region: "${region}") {
                        sh """
                            aws ecr get-login-password --region ${region} | \
                            docker login --username AWS --password-stdin ${ACC_ID}.dkr.ecr.${region}.amazonaws.com

                            docker push ${ACC_ID}.dkr.ecr.${region}.amazonaws.com/roboshop/catalogue:${appVersion}
                        """
                    }
                }
            }
        }
    }

    post {
        always {
            cleanWs()
        }
        success {
            echo "Pipeline success"
        }
        failure {
            echo "Pipeline failed"
        }
    }
}
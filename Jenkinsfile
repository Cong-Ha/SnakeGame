pipeline {
    agent any
    
    environment {
        NODE_VERSION = '18'
        DOCKER_IMAGE_NAME = 'uzomaki/snake-game'
        DOCKER_TAG = "${env.BUILD_NUMBER}"
        DOCKERHUB_CREDENTIALS_ID = 'docker_id'
        DOCKERHUB_CREDENTIALS = credentials('docker_id')
        DOCKERHUB_URL = 'https://registry.hub.docker.com'
        TRIVY_SEVERITY = "HIGH,CRITICAL"
        ZAP_TARGET_URL = "http://172.232.3.70:5000"
    }
    
    stages {
        stage('Clone Repository') {
            steps {
                checkout scm
            }
        }
        
        stage('Static Security Scan') {
            agent {label 'App-Server'}
            steps {
                script {
                    snykSecurity(
                        snykInstallation: 'Snyk',
                        snykTokenId: 'snyk_token',
                        severity: 'critical'
                    )
                }
            }
        }
        
        stage('SonarQube Analysis') {
            agent {label 'App-Server'}
            steps {
                script {
                    def scannerHome = tool 'SonarQube-Scanner'
                    withSonarQubeEnv('sonarqube') {
                        sh "${scannerHome}/bin/sonar-scanner \
                            -Dsonar.projectKey=snake-game \
                            -Dsonar.sources=."
                    }
                }
            }
        }
        
        stage('BUILD-AND-TAG') {
            agent {label 'App-Server'}
            steps {
                script {
                    echo "Building Docker image ${DOCKER_IMAGE_NAME}..."
                    app = docker.build(DOCKER_IMAGE_NAME)
                    app.tag("latest")
                }
            }
        }
        
        stage('Container Vulnerability Scan (Trivy)') {
            agent {label 'App-Server'}
            steps {
                script {
                    echo "Scanning Docker image ${DOCKER_IMAGE_NAME} for vulnerabilities..."
 
                    // Generate JSON report - output to stdout and redirect
                    sh """
                        docker run --rm aquasec/trivy:latest image \
                        --exit-code 0 \
                        --format json \
                        --severity ${TRIVY_SEVERITY} \
                        ${DOCKER_IMAGE_NAME}:latest > trivy-report.json
                        
                        echo "=== Trivy JSON report created ==="
                        ls -la trivy-report.json
                    """
 
                    // Generate HTML report - output to stdout and redirect
                    sh """
                        docker run --rm aquasec/trivy:latest image \
                        --exit-code 0 \
                        --format template \
                        --template "@/contrib/html.tpl" \
                        --severity ${TRIVY_SEVERITY} \
                        ${DOCKER_IMAGE_NAME}:latest > trivy-report.html
                        
                        echo "=== Trivy HTML report created ==="
                        ls -la trivy-report.html
                    """
                    
                    // Summarize vulnerabilities
                    if (fileExists('trivy-report.json')) {
                        def reportContent = readFile('trivy-report.json')
                        def reportJson = new groovy.json.JsonSlurper().parseText(reportContent)
 
                        def highCount = 0
                        def criticalCount = 0
 
                        reportJson.Results.each { result ->
                            result.Vulnerabilities?.each { vuln ->
                                switch (vuln.Severity) {
                                    case 'HIGH': highCount++; break
                                    case 'CRITICAL': criticalCount++; break
                                }
                            }
                        }
 
                        echo "=== TRIVY VULNERABILITY SUMMARY ==="
                        echo "HIGH vulnerabilities: ${highCount}"
                        echo "CRITICAL vulnerabilities: ${criticalCount}"
 
                        if (criticalCount > 0) {
                            echo "⚠️ WARNING: Critical vulnerabilities detected: ${criticalCount}"
                        }
                    }
                }
            }
            post {
                always {
                    echo "Archiving Trivy reports..."
                    archiveArtifacts artifacts: 'trivy-report.json,trivy-report.html', allowEmptyArchive: true
                }
            }
        }
        
        stage('Push Docker Image to Docker Hub') {
            agent {label 'App-Server'}
            steps {
                script {
                    echo "Pushing Docker image ${DOCKER_IMAGE_NAME}:latest to Docker Hub..."
                    docker.withRegistry(env.DOCKERHUB_URL, "${env.DOCKERHUB_CREDENTIALS_ID}") {
                        app.push("latest")
                        app.push("${env.BUILD_NUMBER}")
                    }
                }
            }
        }
        
        stage('Deployment') {
            agent {label 'App-Server'}
            steps{
                echo "Starting deployment using docker-compose..."
                script{
                    dir("$WORKSPACE") {
                        sh '''
                            docker-compose down
                            docker-compose up -d
                            docker ps
                        '''    
                    }
                }
                echo 'Deployment completed successfully'
            }
        }
        
        stage('DAST Scan with OWASP ZAP') {
            agent {label 'App-Server'}
            steps {
                script {
                    echo 'Running OWASP ZAP baseline scan...'

                    // Create a Docker named volume for ZAP reports
                    def volumeName = "zap-reports-${BUILD_NUMBER}"
                    sh "docker volume create ${volumeName}"
                    echo "Created Docker volume: ${volumeName}"

                    // Run ZAP with named volume mount
                    def zapExitCode = sh(script: """
                        docker run --rm --user root --network host \
                        -v ${volumeName}:/zap/wrk:rw \
                        ghcr.io/zaproxy/zaproxy:stable \
                        zap-baseline.py -t ${ZAP_TARGET_URL} \
                        -r zap_report.html \
                        -J zap_report.json
                    """, returnStatus: true)

                    echo "ZAP scan finished with exit code: ${zapExitCode}"

                    // Start a long-running container with the volume mounted to extract files
                    def helperContainerId = sh(script: """
                        docker run -d -v ${volumeName}:/data alpine sleep 300
                    """, returnStdout: true).trim()

                    echo "Helper container ID: ${helperContainerId}"

                    // List files in volume
                    echo "Files in volume:"
                    sh "docker exec ${helperContainerId} ls -la /data/"

                    // Use docker cp to copy files from helper container to Jenkins workspace
                    echo "Copying ZAP reports from container to workspace..."
                    sh """
                        docker cp ${helperContainerId}:/data/zap_report.html ./zap_report.html && echo "HTML copied successfully" || echo "Failed to copy HTML"
                        docker cp ${helperContainerId}:/data/zap_report.json ./zap_report.json && echo "JSON copied successfully" || echo "Failed to copy JSON"
                    """

                    // Clean up helper container and volume
                    sh "docker rm -f ${helperContainerId}"
                    sh "docker volume rm ${volumeName}"
                    echo "Cleaned up resources"

                    // Verify files in workspace
                    echo "Verifying files in workspace:"
                    sh "ls -la ./zap_report.* 2>/dev/null || echo 'No ZAP report files found in workspace'"

                    // Parse ZAP JSON report safely
                    if (fileExists('zap_report.json')) {
                        try {
                            def zapContent = readFile('zap_report.json')
                            def zapJson = new groovy.json.JsonSlurper().parseText(zapContent)

                            def highCount = 0
                            def mediumCount = 0
                            def lowCount = 0

                            zapJson.site.each { site ->
                                site.alerts.each { alert ->
                                    switch (alert.risk) {
                                        case 'High': highCount++; break
                                        case 'Medium': mediumCount++; break
                                        case 'Low': lowCount++; break
                                    }
                                }
                            }

                            echo "=== OWASP ZAP DAST SUMMARY ==="
                            echo "High severity issues: ${highCount}"
                            echo "Medium severity issues: ${mediumCount}"
                            echo "Low severity issues: ${lowCount}"
                            
                            if (highCount > 0) {
                                echo "⚠️ WARNING: High severity security issues detected: ${highCount}"
                            }
                        } catch (Exception e) {
                            echo "Failed to parse ZAP JSON report: ${e.message}"
                            echo "Continuing build..."
                        }
                    } else {
                        echo "ZAP JSON report not found, continuing build..."
                    }
                }
            }
            post {
                always {
                    echo 'Archiving ZAP scan reports...'
                    archiveArtifacts artifacts: 'zap_report.html,zap_report.json', allowEmptyArchive: true
                }
            }
        }
    }
    
    post {
        success {
            echo '=== Pipeline succeeded! ==='
            echo 'Application is running on port 172.232.3.70:5000'
            
            // Publish Trivy HTML report in Jenkins UI
            publishHTML([
                reportDir: '.',
                reportFiles: 'trivy-report.html',
                reportName: 'Trivy Vulnerability Report',
                keepAll: true,
                alwaysLinkToLastBuild: true,
                allowMissing: true
            ])
            
            // Publish ZAP HTML report in Jenkins UI
            publishHTML([
                reportDir: '.',
                reportFiles: 'zap_report.html',
                reportName: 'OWASP ZAP DAST Report',
                keepAll: true,
                alwaysLinkToLastBuild: true,
                allowMissing: true
            ])
        }
        failure {
            echo 'Pipeline failed!'
        }
        always {
            echo 'Pipeline execution completed'
        }
    }
}
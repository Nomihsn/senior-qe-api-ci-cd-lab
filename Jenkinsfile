pipeline {

    agent any

    environment {
        QMETRY_URL = 'https://qtmcloud.qmetry.com/rest/api/automation/importresult'
    }

    stages {

        stage('Test Jenkins') {
            steps {
                echo 'Jenkins is working from GitHub!'
            }
        }

        stage('Run API Tests') {
            steps {
                bat '''
                    if not exist "newman-reports" mkdir "newman-reports"

                    newman run "postman\\Senior QE API Interview Lab.postman_collection.json" ^
                    -e "postman\\API-QA.postman_environment.json" ^
                    -r cli,junit,htmlextra ^
                    --reporter-junit-export "newman-reports\\junit.xml" ^
                    --reporter-htmlextra-export "newman-reports\\newman-report.html"
                '''
            }
        }

        stage('Request QMetry Upload URL') {
            steps {

                withCredentials([
                    string(
                        credentialsId: 'qmetry-api-key',
                        variable: 'QMETRY_API_KEY'
                    )
                ]) {

                    powershell '''
                        $body = @{
                            format = "junit"
                            environment = "QA"
                            build = "$env:BUILD_NUMBER"
                            isZip = $false
                            attachFile = $false
                            matchTestSteps = $false
                        } | ConvertTo-Json -Compress

                        $headers = @{
                            "apiKey" = $env:QMETRY_API_KEY
                        }

                        Write-Host "Requesting QMetry upload URL..."

                        $response = Invoke-RestMethod `
                            -Uri $env:QMETRY_URL `
                            -Method POST `
                            -Headers $headers `
                            -ContentType "application/json" `
                            -Body $body

                        if (-not $response.url) {
                            Write-Error "QMetry did not return an upload URL."
                            exit 1
                        }

                        if (-not $response.trackingId) {
                            Write-Error "QMetry did not return a tracking ID."
                            exit 1
                        }

                        $response | ConvertTo-Json -Depth 10 |
                            Out-File "newman-reports\\qmetry-upload-response.json" -Encoding utf8

                        $response.url |
                            Out-File "newman-reports\\qmetry-upload-url.txt" -Encoding ascii

                        $response.trackingId |
                            Out-File "newman-reports\\qmetry-tracking-id.txt" -Encoding ascii

                        Write-Host "QMetry upload URL received."
                        Write-Host "QMetry tracking ID received."
                    '''
                }
            }
        }

        stage('Upload JUnit Result to QMetry') {
            steps {

                bat '''
                    set /p QMETRY_UPLOAD_URL=<newman-reports\\qmetry-upload-url.txt

                    echo Uploading JUnit result to QMetry...

                    curl.exe --fail --request PUT ^
                        --header "Content-Type: multipart/form-data" ^
                        --upload-file "newman-reports\\junit.xml" ^
                        "%QMETRY_UPLOAD_URL%"

                    if errorlevel 1 (
                        echo QMetry JUnit upload failed.
                        exit /b 1
                    )

                    echo JUnit result uploaded to QMetry successfully.
                '''
            }
        }

        stage('Check QMetry Import Status') {
            steps {

                withCredentials([
                    string(
                        credentialsId: 'qmetry-api-key',
                        variable: 'QMETRY_API_KEY'
                    )
                ]) {

                    powershell '''
                        $trackingId = (Get-Content `
                            "newman-reports\\qmetry-tracking-id.txt" `
                            -Raw).Trim()

                        $trackingUrl = `
                            "https://qtmcloud.qmetry.com/rest/api/automation/importresult/track?trackingId=$trackingId"

                        $headers = @{
                            "apiKey" = $env:QMETRY_API_KEY
                        }

                        Write-Host "Checking QMetry import status..."

                        $maxAttempts = 12
                        $attempt = 0

                        do {

                            $attempt++

                            $response = Invoke-RestMethod `
                                -Uri $trackingUrl `
                                -Method GET `
                                -Headers $headers `
                                -ContentType "application/json"

                            Write-Host "Attempt $attempt"
                            Write-Host "Process Status: $($response.processStatus)"
                            Write-Host "Import Status: $($response.importStatus)"

                            if ($response.importStatus -eq "SUCCESS") {

                                Write-Host "======================================"
                                Write-Host "QMetry import completed successfully."
                                Write-Host "======================================"

                                $response | ConvertTo-Json -Depth 10 |
                                    Out-File `
                                    "newman-reports\\qmetry-import-status.json" `
                                    -Encoding utf8

                                exit 0
                            }

                            if ($response.importStatus -eq "FAILED") {

                                $response | ConvertTo-Json -Depth 10 |
                                    Out-File `
                                    "newman-reports\\qmetry-import-status.json" `
                                    -Encoding utf8

                                Write-Error "QMetry import FAILED."

                                if ($response.detailedMessage) {
                                    Write-Error $response.detailedMessage
                                }

                                exit 1
                            }

                            Start-Sleep -Seconds 5

                        } while ($attempt -lt $maxAttempts)

                        Write-Error "QMetry import did not complete within the expected time."

                        exit 1
                    '''
                }
            }
        }
    }

    post {

        always {

            junit 'newman-reports/junit.xml'

            publishHTML(target: [
                reportDir: 'newman-reports',
                reportFiles: 'newman-report.html',
                reportName: 'Newman API Collection Test Report',
                keepAll: true,
                alwaysLinkToLastBuild: true,
                allowMissing: false
            ])

            archiveArtifacts(
                artifacts: 'newman-reports/qmetry-upload-response.json,newman-reports/qmetry-import-status.json',
                allowEmptyArchive: true
            )
        }
    }
}
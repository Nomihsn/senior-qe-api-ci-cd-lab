
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
                        } | ConvertTo-Json -Compress

                        $headers = @{
                            "apiKey" = $env:QMETRY_API_KEY
                        }

                        Write-Host "Requesting QMetry upload URL..."

                        try {

                            $response = Invoke-RestMethod `
                                -Uri $env:QMETRY_URL `
                                -Method POST `
                                -Headers $headers `
                                -ContentType "application/json" `
                                -Body $body

                            if (-not $response.url) {
                                throw "QMetry did not return an upload URL."
                            }

                            if (-not $response.trackingId) {
                                throw "QMetry did not return a tracking ID."
                            }

                            $response | ConvertTo-Json -Depth 10 |
                                Out-File `
                                "newman-reports\\qmetry-upload-response.json" `
                                -Encoding utf8

                            $response.trackingId |
                                Out-File `
                                "newman-reports\\qmetry-tracking-id.txt" `
                                -Encoding ascii

                            # Store the URL as JSON instead of passing it
                            # through a Windows CMD environment variable.
                            @{
                                url = $response.url
                            } |
                                ConvertTo-Json -Compress |
                                Out-File `
                                "newman-reports\\qmetry-upload-url.json" `
                                -Encoding utf8

                            Write-Host "QMetry upload URL received successfully."
                            Write-Host "QMetry tracking ID received successfully."

                            # Display parameter names only.
                            # Do NOT display parameter values because the URL
                            # contains temporary AWS credentials.
                            Write-Host ""
                            Write-Host "QMetry upload URL parameter names:"

                            $uri = [System.Uri]$response.url

                            $uri.Query.TrimStart('?') -split '&' |
                                ForEach-Object {
                                    ($_ -split '=')[0]
                                } |
                                Sort-Object -Unique |
                                ForEach-Object {
                                    Write-Host " - $_"
                                }
                        }
                        catch {

                            Write-Host "=========================================="
                            Write-Host "QMetry API returned an error"
                            Write-Host "=========================================="

                            Write-Host $_.Exception.Message

                            if ($_.Exception.Response) {

                                try {

                                    $reader = New-Object `
                                        System.IO.StreamReader(
                                            $_.Exception.Response.GetResponseStream()
                                        )

                                    $errorBody = $reader.ReadToEnd()

                                    Write-Host ""
                                    Write-Host "QMetry Response Body:"
                                    Write-Host $errorBody

                                }
                                catch {

                                    Write-Host `
                                        "Could not read QMetry error response body."
                                }
                            }

                            exit 1
                        }
                    '''
                }
            }
        }

        stage('Upload JUnit Result to QMetry') {
            steps {
                withCredentials([
                    string(
                        credentialsId: 'qmetry-api-key',
                        variable: 'QMETRY_API_KEY'
                    )
                ]) {
                    powershell '''
                        Write-Host "Uploading JUnit result to QMetry..."

                        try {

                            # Read the presigned URL directly in PowerShell.
                            # This avoids passing the URL through CMD/bat.
                            $uploadConfig = Get-Content `
                                "newman-reports\\qmetry-upload-url.json" `
                                -Raw |
                                ConvertFrom-Json

                            $uploadUrl = $uploadConfig.url

                            if (-not $uploadUrl) {
                                throw "QMetry upload URL is empty."
                            }

                            # Verify the URL still contains the expected
                            # presigned AWS parameters.
                            $uri = [System.Uri]$uploadUrl

                            $requiredParameters = @(
                                "X-Amz-Algorithm",
                                "X-Amz-Credential",
                                "X-Amz-Date",
                                "X-Amz-Expires",
                                "X-Amz-Security-Token",
                                "X-Amz-Signature",
                                "X-Amz-SignedHeaders"
                            )

                            $queryParameters = @{}

                            $uri.Query.TrimStart('?') -split '&' |
                                ForEach-Object {

                                    $parts = $_ -split '=', 2

                                    if ($parts.Count -eq 2) {
                                        $queryParameters[$parts[0]] = $parts[1]
                                    }
                                }

                            Write-Host ""
                            Write-Host "Validating QMetry presigned URL..."

                            $missingParameters = @()

                            foreach ($parameter in $requiredParameters) {

                                if (-not $queryParameters.ContainsKey($parameter)) {
                                    $missingParameters += $parameter
                                }
                            }

                            if ($missingParameters.Count -gt 0) {

                                Write-Host ""
                                Write-Host "Missing QMetry URL parameters:"

                                foreach ($parameter in $missingParameters) {
                                    Write-Host " - $parameter"
                                }

                                throw `
                                    "QMetry returned an incomplete presigned S3 URL."
                            }

                            Write-Host `
                                "All required AWS presigned URL parameters are present."

                            $junitFile = `
                                (Resolve-Path `
                                "newman-reports\\junit.xml").Path

                            # Read JUnit as raw bytes.
                            $fileBytes = `
                                [System.IO.File]::ReadAllBytes($junitFile)

                            Write-Host ""
                            Write-Host "JUnit file size: $($fileBytes.Length) bytes"

                            # Use HttpWebRequest so the presigned URL is passed
                            # directly without CMD/environment-variable parsing.
                            $request = `
                                [System.Net.HttpWebRequest]::Create($uploadUrl)

                            $request.Method = "PUT"
                            $request.ContentType = "multipart/form-data"
                            $request.ContentLength = $fileBytes.Length

                            $requestStream = $request.GetRequestStream()

                            try {
                                $requestStream.Write(
                                    $fileBytes,
                                    0,
                                    $fileBytes.Length
                                )
                            }
                            finally {
                                $requestStream.Close()
                            }

                            try {

                                $webResponse = $request.GetResponse()

                                $statusCode = `
                                    [int]$webResponse.StatusCode

                                $responseStream = `
                                    $webResponse.GetResponseStream()

                                $reader = `
                                    New-Object System.IO.StreamReader(
                                        $responseStream
                                    )

                                $responseBody = $reader.ReadToEnd()

                                $reader.Close()
                                $responseStream.Close()

                                Write-Host ""
                                Write-Host "=========================================="
                                Write-Host "QMetry Upload Response"
                                Write-Host "=========================================="

                                Write-Host "HTTP_STATUS=$statusCode"

                                if ($responseBody) {
                                    Write-Host $responseBody
                                }

                                $responseBody |
                                    Out-File `
                                    "newman-reports\\qmetry-upload-response.txt" `
                                    -Encoding utf8

                                "$statusCode" |
                                    Out-File `
                                    "newman-reports\\qmetry-http-status.txt" `
                                    -Encoding ascii

                                if ($statusCode -ne 200) {
                                    throw `
                                        "QMetry JUnit upload failed with HTTP $statusCode."
                                }

                            }
                            catch `
                                [System.Net.WebException] {

                                $errorResponse = `
                                    $_.Exception.Response

                                $statusCode = 0
                                $errorBody = ""

                                if ($errorResponse) {

                                    $statusCode = `
                                        [int]$errorResponse.StatusCode

                                    $stream = `
                                        $errorResponse.GetResponseStream()

                                    $reader = `
                                        New-Object System.IO.StreamReader(
                                            $stream
                                        )

                                    $errorBody = $reader.ReadToEnd()

                                    $reader.Close()
                                    $stream.Close()
                                }

                                Write-Host ""
                                Write-Host "=========================================="
                                Write-Host "QMetry Upload Failed"
                                Write-Host "=========================================="

                                Write-Host "HTTP_STATUS=$statusCode"
                                Write-Host $errorBody

                                $errorBody |
                                    Out-File `
                                    "newman-reports\\qmetry-upload-response.txt" `
                                    -Encoding utf8

                                "$statusCode" |
                                    Out-File `
                                    "newman-reports\\qmetry-http-status.txt" `
                                    -Encoding ascii

                                throw
                            }

                            Write-Host ""
                            Write-Host `
                                "QMetry JUnit upload completed successfully."

                        }
                        catch {

                            Write-Host ""
                            Write-Host "QMetry upload error:"
                            Write-Host $_.Exception.Message

                            exit 1
                        }
                    '''
                }
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
                        $trackingId = (
                            Get-Content `
                                "newman-reports\\qmetry-tracking-id.txt" `
                                -Raw
                        ).Trim()

                        $trackingUrl =
                            "https://qtmcloud.qmetry.com/rest/api/automation/importresult/track?trackingId=$trackingId"

                        $headers = @{
                            "apiKey" = $env:QMETRY_API_KEY
                        }

                        Write-Host "Checking QMetry import status..."

                        $maxAttempts = 12
                        $attempt = 0

                        do {

                            $attempt++

                            try {

                                $response = Invoke-RestMethod `
                                    -Uri $trackingUrl `
                                    -Method GET `
                                    -Headers $headers `
                                    -ContentType "application/json"

                                Write-Host ""
                                Write-Host "Attempt: $attempt"
                                Write-Host "Process Status: $($response.processStatus)"
                                Write-Host "Import Status: $($response.importStatus)"
                                Write-Host ""

                                $response |
                                    ConvertTo-Json -Depth 10 |
                                    Out-File `
                                    "newman-reports\\qmetry-import-status.json" `
                                    -Encoding utf8

                                if ($response.importStatus -eq "SUCCESS") {

                                    Write-Host `
                                        "=========================================="

                                    Write-Host `
                                        "QMetry import completed successfully."

                                    Write-Host `
                                        "=========================================="

                                    exit 0
                                }

                                if ($response.importStatus -eq "FAILED") {

                                    Write-Host `
                                        "=========================================="

                                    Write-Host `
                                        "QMetry import FAILED."

                                    Write-Host `
                                        "=========================================="

                                    if ($response.detailedMessage) {

                                        Write-Host "Details:"
                                        Write-Host `
                                            $response.detailedMessage
                                    }

                                    exit 1
                                }

                            }
                            catch {

                                Write-Host `
                                    "Error while checking QMetry status:"

                                Write-Host $_.Exception.Message

                                exit 1
                            }

                            Write-Host `
                                "Import still in progress. Waiting 5 seconds..."

                            Start-Sleep -Seconds 5

                        } while ($attempt -lt $maxAttempts)

                        Write-Error `
                            "QMetry import did not complete within the expected time."

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
                artifacts: '''
                    newman-reports/qmetry-upload-response.json,
                    newman-reports/qmetry-upload-url.json,
                    newman-reports/qmetry-upload-response.txt,
                    newman-reports/qmetry-http-status.txt,
                    newman-reports/qmetry-import-status.json
                ''',
                allowEmptyArchive: true
            )
        }
    }
}

pipeline {
    agent any

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
        }
    }
}
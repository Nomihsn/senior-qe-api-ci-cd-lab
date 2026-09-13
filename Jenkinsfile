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
                    newman run "postman\\Senior QE API Interview Lab.postman_collection.json" ^
                    -e "postman\\API-QA.postman_environment.json"
                '''
            }
        }
    }
}
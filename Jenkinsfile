pipeline {
    agent any

    stages {
        stage('Test Jenkins') {
            steps {
                echo 'Jenkins is working from GitHub!'
            }
        }

        stage('Check Node and Newman') {
            steps {
                bat 'node --version'
                bat 'newman --version'
            }
        }
    }
}
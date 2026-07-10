pipeline {
    agent any

    environment {
        COURSE = 'DevOps'
    }

    options {
        disableConcurrentBuilds()
        timeout(time: 5, unit: 'MINUTES')
    }

    parameters {
        string(
            name: 'BRANCH_NAME',
            defaultValue: 'main',
            description: 'Branch to build'
        )
    }

    stages {
        stage('Build') {
            steps {
                sh '''
                    echo "Building"
                    echo "Course: $COURSE"
                    echo "Branch: $BRANCH_NAME"
                    sleep 10
                '''
            }
        }

        stage('Test') {
            steps {
                sh '''
                    echo "Testing"
                '''
            }
        }

        stage('Deploy') {
            steps {
                sh '''
                    echo "Deploying"
                '''
            }
        }
    }
}
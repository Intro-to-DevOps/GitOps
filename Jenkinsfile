pipeline {
    agent any

    stages {
        stage('Lint') {
            steps {
                sh 'helm lint .'
            }
        }
        stage('Package') {
            steps {
                sh 'helm package .'
            }
        }
        // Additional stages for deploying or publishing chart would go here
    }
}

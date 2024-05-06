pipeline {
    agent any

// tools {
//     maven 'Maven3'
//     jdk 'Java17'
// }

    options {
        skipDefaultCheckout()
    }

    stages {

        stage('Test branch pipeline') {

            when {
                branch 'test'
                beforeAgent true
                beforeOptions true
            }

            stages {
                stage('Git checkout') {
                    steps {
                        checkout changelog: false, poll: false, scm: scmGit(branches: [[name: 'test']], extensions: [], userRemoteConfigs: [[credentialsId: 'github_credentials', url: 'https://github.com/MedEzzedine/Ecommerce-Microservices']])
                    }
                }
                stage('Build docker image') {
                    steps {
                        echo "Build docker image"
                    }
                }

                stage('Vulnerability scan') {
                    steps {
                        echo "Vulnerability scan"
                    }
                }

                stage('Push to Dockerhub') {
                    steps {
                        echo "Push to Dockerhub"
                    }
                }

                stage('Kubectl apply new deployment') {
                    steps {
                        echo "Kubectl apply new deployment"
                    }
                }

                stage('Integration testing') {
                    steps {
                        echo "Integration testing"
                    }
                }
            }
        }
    }
}


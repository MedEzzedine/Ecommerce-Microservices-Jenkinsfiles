def microservices = ['ecomm-cart', 'ecomm-product', 'ecomm-order', 'ecomm-user']

pipeline {
    agent any

    tools {
        maven "maven3"
    }

    options {
        skipDefaultCheckout()
    }

    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        GITHUB_REPO = 'https://github.com/MedEzzedine/Ecommerce-Microservices.git'
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
                stage('Docker Login') {
                    steps {
                        
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


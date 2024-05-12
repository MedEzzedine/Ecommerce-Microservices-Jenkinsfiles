pipeline {
    agent none

// tools {
//     maven 'Maven3'
//     jdk 'Java17'
// }

    options {
        skipDefaultCheckout()
    }

    stages {
     
        stage('Feature PR to dev') {
            when {
                changeRequest target: 'dev/*', comparator: "GLOB"
                changeRequest branch: 'feature/*', comparator: "GLOB"
                not { changeRequest branch: 'feature/frontend*', comparator: "GLOB" }
                beforeAgent true
                beforeOptions true
            }

            agent any

            stages {

                stage('Git checkout') {
                        steps {
                            checkout changelog: false, poll: false, scm: scmGit(branches: [[name: env.CHANGE_BRANCH]], extensions: [], userRemoteConfigs: [[credentialsId: 'github_credentials', url: 'https://github.com/MedEzzedine/Ecommerce-Microservices']])
                        }
                    }
                    
                stage('Compile') {
                    steps {
                        echo "Compile"
                    }
                }
                stage('Unit testing') {
                    steps {
                        echo "Unit testing"
                    }
                }
                stage('Build Stage') {
                    steps {
                        echo "Building"
                    }
                }
                stage('SonarQube SAST') {
                    steps {
                        echo "SonarQube SAST"
                    }
                }
                stage('Quality Gates') {
                    steps {
                        echo "Quality "
                    }
                }
            }
        }

        

        stage('Frontend Feature PR to dev') {
            when {
                changeRequest target: 'dev/*', comparator: "GLOB"
                changeRequest branch: 'feature/frontend*', comparator: "GLOB"
                beforeAgent true
                beforeOptions true
            }

            agent any

            stages {

                stage('Git checkout') {
                        steps {
                            checkout changelog: false, poll: false, scm: scmGit(branches: [[name: env.CHANGE_BRANCH]], extensions: [], userRemoteConfigs: [[credentialsId: 'github_credentials', url: 'https://github.com/MedEzzedine/Ecommerce-Microservices']])
                        }
                    }
                    
                stage('Compile') {
                    steps {
                        echo "Compile"
                    }
                }
                stage('Unit testing') {
                    steps {
                        echo "Unit testing"
                    }
                }
                stage('Build Stage') {
                    steps {
                        echo "Building"
                    }
                }
            }
        }
    }
}


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
        DOCKERHUB_CREDENTIALS_ID = 'docker_credentials'
        DOCKERHUB_USER = 'medez'
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
                        withCredentials([usernamePassword(credentialsId: DOCKERHUB_CREDENTIALS_ID, usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]) {
                            sh "docker login -u ${USERNAME} -p ${PASSWORD}"
                        }
                    }
                }

                //TODO: Retrieve jars from Nexus/JFrog
                stage('Build jars') {
                    steps {
                        script {
                            for(def microservice in microservices) {
                                dir("micro-services/${microservice}") {
                                    sh "mvn clean install"
                                }
                            }
                        }
                    }
                }

                stage('Build Docker images') {
                    steps {
                        script {
                            for(def microservice in microservices) {
                                dir("micro-services/${microservice}") {
                                    sh "docker build -t $DOCKERHUB_USER/$microservice:$BRANCH_NAME-$BUILD_NUMBER ."
                                }
                            }
                            // TODO: Add arguments when building frontend image
                            dir('frontend') {
                                // "--network=host" to avoid DNS problem while running npm ci
                                sh "docker build -t $DOCKERHUB_USER/ecomm-frontend:$BRANCH_NAME-$BUILD_NUMBER -f Dockerfile.local --network=host ."
                            }
                        }
                    }
                }

                stage('Vulnerability scan') {
                    steps {
                        script {
                            for (def microservice in microservices) {
                                // -q: quiet mode (avoid unnecessary output), --severity CRITICAL exit code will be 1 when a CRITICAL vulnerability is found
                                // TODO: Add back --exit-code 1 for the final pipeline
                                sh(script: 'docker run --rm aquasec/trivy image $DOCKERHUB_USER/$microservice:$BRANCH_NAME-$BUILD_NUMBER | awk \'{print "<tr><td>" $1 "</td><td>" $2 "</td><td>" $3 "</td></tr>"}\' > trivy-report-${microservice}.html', returnStdout: true)
                                archiveArtifacts "trivy-report-${microservice}.html", allowEmptyArchive: true
                            }
                            
                            sh(script: 'docker run --rm aquasec/trivy image $DOCKERHUB_USER/ecomm-frontend:$BRANCH_NAME-$BUILD_NUMBER | awk \'{print "<tr><td>" $1 "</td><td>" $2 "</td><td>" $3 "</td></tr>"}\' > trivy-report-ecomm-frontend.html', returnStdout: true)
                            archiveArtifacts "trivy-report-ecomm-frontend.html", allowEmptyArchive: true
                        }
                    }
                }

                stage('Push to Dockerhub') {
                    steps {
                        script {
                            for (microservice in microservices) {
                                sh "docker push $DOCKERHUB_USER/$microservice:$BRANCH_NAME-$BUILD_NUMBER"
                            }
                            sh "docker push $DOCKERHUB_USER/ecomm-frontend:$BRANCH_NAME-$BUILD_NUMBER"
                        }
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

    post {
        always {
            script {
                sh 'docker logout'
                echo 'Logged out from DockerHub successfully.'
            }
        }
        
        // failure {
        //     slackSend color: "danger", message: "Backend pipeline failed."
        // }
    }
}


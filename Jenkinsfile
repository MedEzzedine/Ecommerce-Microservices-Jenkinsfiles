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
        K8S_MASTER_HOST = 'k8s.dev.mohamedezzedine.me' // To be changed with a fixed url
        K8S_MASTER_SSH_CREDENTIALS_ID = 'k8s-master-ssh'
        K8S_MASTER_SSH_USER = 'ubuntu'
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
                                    sh "docker tag $DOCKERHUB_USER/$microservice:$BRANCH_NAME-$BUILD_NUMBER $DOCKERHUB_USER/$microservice:latest"
                                }
                            }
                            // TODO: Add arguments when building frontend image
                            dir('frontend') {
                                // "--network=host" to avoid DNS problem while running npm ci
                                sh "docker build -t $DOCKERHUB_USER/ecomm-frontend:$BRANCH_NAME-$BUILD_NUMBER -f Dockerfile.local --network=host ."
                                sh "docker tag $DOCKERHUB_USER/ecomm-frontend:$BRANCH_NAME-$BUILD_NUMBER $DOCKERHUB_USER/ecomm-frontend:latest"
                            }
                        }
                    }
                }

                stage('Vulnerability scan') {
                    steps {
                        script {
                            
                            sh "curl -o $PWD/html.tpl https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/html.tpl"

                            for (def microservice in microservices) {
                                // -q: quiet mode (avoid unnecessary output), --severity CRITICAL exit code will be 1 when a CRITICAL vulnerability is found
                                // TODO: Add back --exit-code 1 for the final pipeline
                                sh "docker run --rm -v /var/run/docker.sock:/var/run/docker.sock -v $PWD:/root/.cache/ -v $PWD:/tmp/.cache -v $PWD/html.tpl:/tmp/html.tpl aquasec/trivy image --scanners vuln --format template --template '@/tmp/html.tpl' $DOCKERHUB_USER/${microservice}:$BRANCH_NAME-$BUILD_NUMBER > trivy-report-${microservice}.html"
                                archiveArtifacts artifacts: "trivy-report-${microservice}.html", allowEmptyArchive: true
                            }
                            
                            sh "docker run --rm -v /var/run/docker.sock:/var/run/docker.sock -v $PWD:/root/.cache/ -v $PWD/html.tpl:/tmp/html.tpl aquasec/trivy image --scanners vuln --format template --template '@/tmp/html.tpl' $DOCKERHUB_USER/ecomm-frontend:$BRANCH_NAME-$BUILD_NUMBER > trivy-report-ecomm-frontend.html"
                            archiveArtifacts artifacts: "trivy-report-ecomm-frontend.html", allowEmptyArchive: true

                            sh "rm $PWD/html.tpl"
                        }
                    }
                }

                stage('Push to Dockerhub') {
                    steps {
                        script {
                            for (microservice in microservices) {
                                sh "docker push $DOCKERHUB_USER/$microservice:$BRANCH_NAME-$BUILD_NUMBER"
                                sh "docker push $DOCKERHUB_USER/$microservice:latest"
                            }
                            sh "docker push $DOCKERHUB_USER/ecomm-frontend:$BRANCH_NAME-$BUILD_NUMBER"
                            sh "docker push $DOCKERHUB_USER/ecomm-frontend:latest"
                        }
                    }
                }

                stage('minikube kubectl -- apply -n test new deployment') {
                    steps {
                        sshagent(credentials: [K8S_MASTER_SSH_CREDENTIALS_ID]) {
                            script {
                                sh '''
                                    [ -d ~/.ssh ] || mkdir ~/.ssh && chmod 0700 ~/.ssh
                                    ssh-keyscan -t rsa,dsa ${K8S_MASTER_HOST} >> ~/.ssh/known_hosts

                                    ssh ${K8S_MASTER_SSH_USER}@${K8S_MASTER_HOST} "kubectl get nodes"

                                    scp -r manifests ${K8S_MASTER_SSH_USER}@${K8S_MASTER_HOST}:~/manifests

                                    ssh ${K8S_MASTER_SSH_USER}@${K8S_MASTER_HOST} "minikube kubectl -- apply -n test -f manifests/test-env/infrastructure/configMap.yml"
                                    ssh ${K8S_MASTER_SSH_USER}@${K8S_MASTER_HOST} "minikube kubectl -- apply -n test -f manifests/test-env/infrastructure/postgres.yml"
                                    ssh ${K8S_MASTER_SSH_USER}@${K8S_MASTER_HOST} "minikube kubectl -- apply -n test -f manifests/test-env/infrastructure/redis.yml"
                                    ssh ${K8S_MASTER_SSH_USER}@${K8S_MASTER_HOST} "minikube kubectl -- apply -n test -f manifests/test-env/infrastructure/volume.yml"
                                    sleep 60
                                    ssh ${K8S_MASTER_SSH_USER}@${K8S_MASTER_HOST} "minikube kubectl -- apply -n test -f manifests/test-env/infrastructure/elasticsearch.yml"
                                    sleep 5
                                    ssh ${K8S_MASTER_SSH_USER}@${K8S_MASTER_HOST} "minikube kubectl -- apply -n test -f manifests/test-env/micro-services/cart.yml"
                                    ssh ${K8S_MASTER_SSH_USER}@${K8S_MASTER_HOST} "minikube kubectl -- apply -n test -f manifests/test-env/micro-services/product.yml"
                                    ssh ${K8S_MASTER_SSH_USER}@${K8S_MASTER_HOST} "minikube kubectl -- apply -n test -f manifests/test-env/micro-services/order.yml"
                                    ssh ${K8S_MASTER_SSH_USER}@${K8S_MASTER_HOST} "minikube kubectl -- apply -n test -f manifests/test-env/micro-services/user.yml"
                                    ssh ${K8S_MASTER_SSH_USER}@${K8S_MASTER_HOST} "minikube kubectl -- apply -n test -f manifests/test-env/micro-services/frontend.yml"
                                '''
                            }
                        }
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


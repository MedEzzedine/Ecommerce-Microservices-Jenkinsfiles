def microservices = ['ecomm-cart', 'ecomm-product', 'ecomm-order', 'ecomm-user']

pipeline {
    agent any

    tools {
        maven "maven3"
        nodejs "Node20"
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
                            dir("edge-services/ecomm-gateway") {
                                sh "mvn clean install"
                            }
                        }
                    }
                }

                stage('Build frontend') {
                    steps {
                        dir("frontend") {
                            sh "npm ci"

                            sh "echo 'REACT_APP_SERVER_BASE_URL=http://gateway:8091' > .env"
                            sh "echo 'REACT_APP_WS_BASE_URL=ws://frontend:80' >> .env"
                            sh "echo 'NODE_ENV=production' >> .env"
                            sh "echo 'REACT_APP_SERVER_BASE_URL=http://frontend:80' >> .env"

                            sh "CI=false npm run build"
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
                            dir("edge-services/ecomm-gateway") {
                                sh "docker build -t $DOCKERHUB_USER/ecomm-gateway:$BRANCH_NAME-$BUILD_NUMBER ."
                                sh "docker tag $DOCKERHUB_USER/ecomm-gateway:$BRANCH_NAME-$BUILD_NUMBER $DOCKERHUB_USER/ecomm-gateway:latest"
                            }

                            dir('frontend') {
                                // "--network=host" to avoid DNS problem while running npm ci
                                sh "docker build -t $DOCKERHUB_USER/ecomm-frontend:$BRANCH_NAME-$BUILD_NUMBER --network=host ."
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
                            for (def microservice in microservices) {
                                sh "docker push $DOCKERHUB_USER/$microservice:$BRANCH_NAME-$BUILD_NUMBER"
                                sh "docker push $DOCKERHUB_USER/$microservice:latest"
                            }

                            // Pushing frontend
                            sh "docker push $DOCKERHUB_USER/ecomm-frontend:$BRANCH_NAME-$BUILD_NUMBER"
                            sh "docker push $DOCKERHUB_USER/ecomm-frontend:latest"

                            // Pushing gateway
                            sh "docker push $DOCKERHUB_USER/ecomm-gateway:$BRANCH_NAME-$BUILD_NUMBER"
                            sh "docker push $DOCKERHUB_USER/ecomm-gateway:latest"
                        }   
                    }
                }


                stage('Scan K8s cluster with kube-bench') {
                    steps {
                        sshagent(credentials: [K8S_MASTER_SSH_CREDENTIALS_ID]) {

                            sh "[ -d ~/.ssh ] || mkdir ~/.ssh && chmod 0700 ~/.ssh"
                            sh "ssh-keyscan -t rsa,dsa ${K8S_MASTER_HOST} >> ~/.ssh/known_hosts"

                            sh "ssh ${K8S_MASTER_SSH_USER}@${K8S_MASTER_HOST} 'sudo kube-bench > kubebench_CIS_${env.BRANCH_NAME}.txt'"
                            sh "scp ${K8S_MASTER_SSH_USER}@${K8S_MASTER_HOST}:~/kubebench_CIS_${env.BRANCH_NAME}.txt ."
                            sh "ssh ${K8S_MASTER_SSH_USER}@${K8S_MASTER_HOST} 'sudo rm kubebench_CIS_${env.BRANCH_NAME}.txt'"

                        }
                    }
                }


                stage('Deploy to K8s test env') {
                    steps {
                        sshagent(credentials: [K8S_MASTER_SSH_CREDENTIALS_ID]) {
                            script {
                                sh '''
                                    ssh ${K8S_MASTER_SSH_USER}@${K8S_MASTER_HOST} rm -rf ~/manifests/test-env
                                    ssh ${K8S_MASTER_SSH_USER}@${K8S_MASTER_HOST} mkdir -p ~/manifests/test-env
                                    scp -r manifests/test-env ${K8S_MASTER_SSH_USER}@${K8S_MASTER_HOST}:~/manifests/test-env
                                    scp scripts/deploy-manifests-test.sh ${K8S_MASTER_SSH_USER}@${K8S_MASTER_HOST}:~/scripts/deploy-manifests-test.sh

                                    ssh ${K8S_MASTER_SSH_USER}@${K8S_MASTER_HOST} chmod +x scripts/deploy-manifests-test.sh
                                    ssh ${K8S_MASTER_SSH_USER}@${K8S_MASTER_HOST} sh scripts/deploy-manifests-test.sh
                                    ssh ${K8S_MASTER_SSH_USER}@${K8S_MASTER_HOST} kubectl get all -n test
                                '''
                            }
                        }
                    }
                }

                stage('Scan manifests with Kubescan') {
                    steps {
                        sshagent(credentials: [K8S_MASTER_SSH_CREDENTIALS_ID]) {
                            script {
                                sh "ssh ${K8S_MASTER_SSH_USER}@${K8S_MASTER_HOST} 'kubescape scan manifests/test-env/infrastructure/*.yml -v > kubescape_infrastructure_test.txt'"
                                sh "scp ${K8S_MASTER_SSH_USER}@${K8S_MASTER_HOST}:~/kubescape_infrastructure_test.txt ."
                                sh "ssh ${K8S_MASTER_SSH_USER}@${K8S_MASTER_HOST} 'kubescape scan manifests/test-env/micro-services/*.yml -v > kubescape_microservices_test.txt'"
                                sh "scp ${K8S_MASTER_SSH_USER}@${K8S_MASTER_HOST}:~/kubescape_microservices_test.txt ."
                                sh "ssh ${K8S_MASTER_SSH_USER}@${K8S_MASTER_HOST} rm -f kubescape_infrastructure_test.txt"
                                sh "ssh ${K8S_MASTER_SSH_USER}@${K8S_MASTER_HOST} rm -f kubescape_microservices_test.txt"
                            }
                        }
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
                slackUploadFile filePath: '**/trufflehog.txt',  initialComment: 'Check TruffleHog Reports!'
                slackUploadFile filePath: '**/trivy-*.txt', initialComment: 'Check Trivy Reports!'
                slackUploadFile filePath: '**/kubebench_CIS_*.txt', initialComment: 'Check Kube-bench Reports!'
                slackUploadFile filePath: '**/kubescape_*.txt', initialComment: 'Check Kube-bench Reports!'
                
            }
        }
        
        success {
            slackSend color: "good", message: "¨Pipeline $BRANCH_NAME-$BUILD_NUMBER succeeded."
        }

        failure {
             slackSend color: "danger", message: "Pipeline $BRANCH_NAME-$BUILD_NUMBER failed."
        }
    }
}


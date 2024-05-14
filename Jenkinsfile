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
     
        stage('Feature PR to dev') {
            when {
                changeRequest target: 'dev/*', comparator: "GLOB"
                changeRequest branch: 'feature/*', comparator: "GLOB"
                not { changeRequest branch: 'feature/frontend*', comparator: "GLOB" }
                beforeAgent true
                beforeOptions true
            }

            stages {

                stage('Git checkout') {
                        steps {
                            checkout changelog: false, poll: false, scm: scmGit(branches: [[name: env.CHANGE_BRANCH]], extensions: [], userRemoteConfigs: [[credentialsId: 'github_credentials', url: GITHUB_REPO]])
                        }
                    }
                    
                stage('Compile') {
                    steps {
                        script {
                            def changes = getPRChangeLog(env.CHANGE_TARGET)
                            for(def microservice in microservices) {
                                if(changes.contains(microservice)) {
                                    dir("micro-services/${microservice}") {
                                        echo "Compiling microservice: ${microservice}"
                                        sh "mvn clean compile"
                                    }
                                }
                            }
                        }
                    }
                }

                stage('Finding Git secrets') {
                    steps {
                        sh "docker run --rm trufflesecurity/trufflehog github --repo=${GITHUB_REPO} --json | tee trufflehog.json"

                        script {
                            def jsonReport = readFile('trufflehog.json')
                            
                            def htmlReport = """
                            <html>
                            <head>
                                <title>Trufflehog Scan Report</title>
                            </head>
                            <body>
                                <h1>Trufflehog Scan Report</h1>
                                <pre>${jsonReport}</pre>
                            </body>
                            </html>
                            """
                            
                            writeFile file: 'scanresults/trufflehog-report.html', text: htmlReport
                        }
        
                        archiveArtifacts artifacts: 'scanresults/trufflehog-report.html', allowEmptyArchive: true
                    }
                }

                stage('Unit testing') {
                    steps {
                        script {
                            def changes = getPRChangeLog(env.CHANGE_TARGET)
                            for(def microservice in microservices) {
                                if(changes.contains(microservice)) {
                                    dir("micro-services/${microservice}") {
                                        echo "Unit testing microservice: ${microservice}"
                                        sh "mvn test"
                                    }
                                }
                            }
                        }
                    }
                }

                stage('SonarQube Analysis') {
                    steps {
                        withSonarQubeEnv('sonar') {
                            script {
                                def changes = getPRChangeLog(env.CHANGE_TARGET)
                                for(def microservice in microservices) {
                                    if(changes.contains(microservice)) {
                                        dir("micro-services/${microservice}") {
                                            echo "Static analysis of microservice: ${microservice}"
                                            sh  "${SCANNER_HOME}/bin/sonar-scanner -Dsonar.projectName=${microservice} -Dsonar.projectKey=${microservice} -Dsonar.java.binaries=."
                                        }
                                    }
                                }
                            }
                        }
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

def getPRChangeLog(target) {
    return sh(
            script: "git --no-pager diff origin/${target} --name-only",
            returnStdout: true
    )//.split('\n')
}

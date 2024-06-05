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

                stage('Finding Git secrets with Trufflehog') {
                    steps {
                        script {

                            sh "docker run -i --rm trufflesecurity/trufflehog github --repo=${GITHUB_REPO} --json > trufflehog.json 2>&1"

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

                stage('Unit testing with JUnit') {
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

                stage('OWASP Dependency Check') {
                    steps {
                        script {
                            def changes = getPRChangeLog(env.CHANGE_TARGET)
                            for(def microservice in microservices) {
                                if(changes.contains(microservice)) {
                                    dir("micro-services/${microservice}") {
                                        echo "Dependency check on microservice: ${microservice}"

                                        dependencyCheck additionalArguments: "-s . -f 'HTML' --project ${microservice}", odcInstallation: 'owasp-dc'
                                    
                                        archiveArtifacts artifacts: "dependency-check-report.html"
                                        
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
                
                stage('Finding Git secrets with Trufflehog') {
                    steps {
                        script {

                            sh "docker run -i --rm trufflesecurity/trufflehog github --repo=${GITHUB_REPO} --json > trufflehog.json 2>&1"

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

                stage('Install NPM Dependencies') {
                    steps {
                        script {
                            dir("frontend") {
                                echo "Installing dependencies..."
                                sh "npm ci"
                            }
                        }
                    }
                }

                stage('Testing with Jest') {
                    steps {
                        script {
                            dir("frontend") {
                                echo "Jest testing..."
                                //sh "npm run test"
                            }
                        }
                    }
                }

                stage('OWASP Dependency Check') {
                    steps {
                        script {
                            dir("frontend") {
                                echo "Dependency check on frontend..."

                                dependencyCheck additionalArguments: "-s . -f 'HTML' --project frontend", odcInstallation: 'owasp-dc'
                            
                                archiveArtifacts artifacts: "dependency-check-report.html"
                            }
                        }
                    }
                }

                stage('SonarQube Analysis') {
                    steps {
                        withSonarQubeEnv('sonar') {
                            script {
                                dir("frontend") {
                                    echo "Static analysis of frontend..."
                                    sh  "${SCANNER_HOME}/bin/sonar-scanner"
                                }
                            }
                        }
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

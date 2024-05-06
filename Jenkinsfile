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
                branch 'main'
                beforeAgent true
                beforeOptions true
            }

            stages {
                stage('Git checkout') {
                    steps {
                        checkout changelog: false, poll: false, scm: scmGit(branches: [[name: 'main']], extensions: [], userRemoteConfigs: [[credentialsId: 'github_credentials', url: 'https://github.com/MedEzzedine/Ecommerce-Microservices']])
                    }
                }

                stage("Update deployment version in manifests for ArgoCD") {
                    steps {
                        echo "Update manifests for ArgoCD"
                    }
                }
            }
        }
    }
}


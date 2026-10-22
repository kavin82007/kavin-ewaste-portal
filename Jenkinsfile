pipeline {
    agent any

    parameters {
        string(name: 'IMAGE_TAG', defaultValue: 'v1', description: 'Image tag to deploy')
        booleanParam(name: 'BUILD_AND_PUSH', defaultValue: true, description: 'Build and push the Docker image')
    }

    environment {
        IMAGE_NAME = 'blackwolf82007/kavin-ewaste-portal'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build and Test') {
            parallel {
                stage('HTML Check') {
                    steps {
                        bat 'findstr /C:"E-Waste Collection" index.html'
                    }
                }
                stage('Dockerfile Check') {
                    steps {
                        bat 'findstr /C:"FROM nginx" Dockerfile'
                    }
                }
            }
        }

        stage('Archive Artifacts') {
            steps {
                archiveArtifacts artifacts: 'index.html, Dockerfile, deployment.yaml', fingerprint: true
            }
        }

        stage('Docker Build and Push') {
            when {
                expression { params.BUILD_AND_PUSH }
            }
            steps {
                bat "docker build -t %IMAGE_NAME%:%IMAGE_TAG% ."
                withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'DH_USER', passwordVariable: 'DH_PASS')]) {
                    bat 'echo %DH_PASS%| docker login -u %DH_USER% --password-stdin'
                    bat "docker push %IMAGE_NAME%:%IMAGE_TAG%"
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                withCredentials([file(credentialsId: 'kubeconfig-file', variable: 'KUBECONFIG')]) {
                    bat 'kubectl apply -f deployment.yaml'
                    bat "kubectl set image deployment/ewaste-portal ewaste-portal=%IMAGE_NAME%:%IMAGE_TAG%"
                    script {
                        def rc = bat(returnStatus: true, script: 'kubectl rollout status deployment/ewaste-portal --timeout=60s')
                        if (rc != 0) {
                            echo 'Rollout failed. Rolling back to the previous stable release.'
                            bat 'kubectl rollout undo deployment/ewaste-portal'
                            bat 'kubectl rollout status deployment/ewaste-portal --timeout=60s'
                            error('Faulty release detected and rolled back')
                        }
                    }
                }
            }
        }
    }
}
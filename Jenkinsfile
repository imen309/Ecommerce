
def microservices = ['ecomm-cart']

pipeline {
    agent any

    tools{
            maven 'maven'
        }

    environment {
        DOCKERHUB_USERNAME = "imenmettichi"
        // Ensure Docker credentials are stored securely in Jenkins
    }

    stages {
        stage('Checkout') {
            steps {
                // Checkout the repository from GitHub
                checkout([
                    $class: 'GitSCM',
                    branches: [[name: env.BRANCH_NAME]], // Checkout the current branch
                    userRemoteConfigs: [[url: 'https://github.com/imen309/Ecommerce.git']]
                ])
            }
        }

        stage('Check Git Secrets') {
            when {
                expression { (env.BRANCH_NAME == 'dev') || (env.BRANCH_NAME == 'test') || (env.BRANCH_NAME == 'master') }
            }
            steps {
                script {
                    // Check each microservice for secrets
                    for (def service in microservices) {
                        dir(service) {
                            // Run TruffleHog to check for secrets in the repository
                            sh 'docker run --rm gesellix/trufflehog --json https://github.com/imen309/Ecommerce.git > trufflehog.json'
                            sh 'cat trufflehog.json' // Output the results
                        }
                    }
                }
            }
        }

        stage('Maven Build') {
            when {
                expression { (env.BRANCH_NAME == 'dev') || (env.BRANCH_NAME == 'test') || (env.BRANCH_NAME == 'master') }
            }
            steps {
                script {
                    // Build each microservice using Maven
                    for (def service in microservices) {
                        dir(service) {
                            sh 'mvn clean install'
                        }
                    }
                }
            }
        }

        stage('Unit Test') {
            when {
                expression { (env.BRANCH_NAME == 'dev') || (env.BRANCH_NAME == 'test') || (env.BRANCH_NAME == 'master') }
            }
            steps {
                script {
                    // Run unit tests for each microservice using Maven
                    for (def service in microservices) {
                        dir(service) {
                            sh 'mvn test'
                        }
                    }
                }
            }
        }
/*
        stage('SonarQube Analysis') {
            when {
                expression { (env.BRANCH_NAME == 'dev') || (env.BRANCH_NAME == 'test') || (env.BRANCH_NAME == 'master') }
            }
            steps {
                script {
                    // Perform static analysis with SonarQube for each microservice
                    for (def service in microservices) {
                        dir(service) {
                            withSonarQubeEnv(credentialsId: 'sonarqube-id') {
                                sh 'mvn sonar:sonar'
                            }
                        }
                    }
                }
            }
        }
*/
        stage('Docker Login') {
            when {
                expression { (env.BRANCH_NAME == 'dev') || (env.BRANCH_NAME == 'test') || (env.BRANCH_NAME == 'master') }
            }
            steps {
                script {
                    // Log into Docker Hub using Jenkins credentials
                    withCredentials([usernamePassword(credentialsId: 'dockerhub', usernameVariable: 'DOCKER_USERNAME', passwordVariable: 'DOCKER_PASSWORD')]) {
                        sh "docker login -u $DOCKER_USERNAME -p $DOCKER_PASSWORD"
                    }
                }
            }
        }

        stage('Docker Build') {
            when {
                expression { (env.BRANCH_NAME == 'dev') || (env.BRANCH_NAME == 'test') || (env.BRANCH_NAME == 'master') }
            }
            steps {
                script {
                    // Build Docker images for each microservice based on the branch
                    for (def service in microservices) {
                        dir(service) {
                            if (env.BRANCH_NAME == 'test') {
                                sh "docker build -t ${DOCKERHUB_USERNAME}/${service}_test:latest ."
                            } else if (env.BRANCH_NAME == 'master') {
                                sh "docker build -t ${DOCKERHUB_USERNAME}/${service}_prod:latest ."
                            } else if (env.BRANCH_NAME == 'dev') {
                                sh "docker build -t ${DOCKERHUB_USERNAME}/${service}_dev:latest ."
                            }
                        }
                    }
                }
            }
        }

        stage('Trivy Image Scan') {
            when {
                expression { (env.BRANCH_NAME == 'dev') || (env.BRANCH_NAME == 'test') || (env.BRANCH_NAME == 'master') }
            }
            steps {
                script {
                    // Scan each Docker image for vulnerabilities using Trivy
                    for (def service in microservices) {
                        def trivyReportFile = "trivy-${service}.txt"
                        if (env.BRANCH_NAME == 'test') {
                            sh "docker run --rm -v /var/run/docker.sock:/var/run/docker.sock -v $PWD:/tmp/.cache/ aquasec/trivy image --scanners vuln --timeout 30m ${DOCKERHUB_USERNAME}/${service}_test:latest > ${trivyReportFile}"
                        } else if (env.BRANCH_NAME == 'master') {
                            sh "docker run --rm -v /var/run/docker.sock:/var/run/docker.sock -v $PWD:/tmp/.cache/ aquasec/trivy image --scanners vuln --timeout 30m ${DOCKERHUB_USERNAME}/${service}_prod:latest > ${trivyReportFile}"
                        } else if (env.BRANCH_NAME == 'dev') {
                            sh "docker run --rm -v /var/run/docker.sock:/var/run/docker.sock -v $PWD:/tmp/.cache/ aquasec/trivy image --scanners vuln --timeout 30m ${DOCKERHUB_USERNAME}/${service}_dev:latest > ${trivyReportFile}"
                        }
                    }
                }
            }
        }

        stage('Docker Push') {
            when {
                expression { (env.BRANCH_NAME == 'dev') || (env.BRANCH_NAME == 'test') || (env.BRANCH_NAME == 'master') }
            }
            steps {
                script {
                    // Push each Docker image to Docker Hub based on the branch
                    for (def service in microservices) {
                        if (env.BRANCH_NAME == 'test') {
                            sh "docker push ${DOCKERHUB_USERNAME}/${service}_test:latest"
                        } else if (env.BRANCH_NAME == 'master') {
                            sh "docker push ${DOCKERHUB_USERNAME}/${service}_prod:latest"
                        } else if (env.BRANCH_NAME == 'dev') {
                            sh "docker push ${DOCKERHUB_USERNAME}/${service}_dev:latest"
                        }
                    }
                }
            }
        }
    }
}

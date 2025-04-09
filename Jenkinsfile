pipeline {
    agent any

    environment {
        // ... (keep your existing environment variables)
    }

    stages {
        stage('Determine Build Branches') {
            steps {
                script {
                    // ... (keep your existing logic)
                }
            }
        }

        stage('Clone and Build Docker Images') {
            steps {
                script {
                    def branches = env.BUILD_BRANCHES.split(',')
                    // Use PowerShell for timestamp on Windows
                    def timestamp = powershell(script: '[DateTime]::Now.ToString("yyyyMMdd-HHmmss")', returnStdout: true).trim()
                    env.IMAGE_TAG = timestamp

                    withCredentials([usernamePassword(
                        credentialsId: env.DOCKER_CRED_ID,
                        usernameVariable: 'DOCKER_USERNAME',
                        passwordVariable: 'DOCKER_PASSWORD'
                    )]) {
                        // Use bat for Docker commands on Windows
                        bat """
                            echo %DOCKER_PASSWORD% | docker login -u %DOCKER_USERNAME% --password-stdin
                        """

                        branches.each { branch ->
                            dir("frontend-${branch}") {
                                git url: "${env.FRONTEND_REPO}", branch: branch, credentialsId: "${env.GIT_CRED_ID}"
                                bat """
                                    docker build -t ${env.IMAGE_REGISTRY}/frontend:${branch}-${env.IMAGE_TAG} .
                                    docker push ${env.IMAGE_REGISTRY}/frontend:${branch}-${env.IMAGE_TAG}
                                """
                            }

                            dir("backend-${branch}") {
                                git url: "${env.BACKEND_REPO}", branch: branch, credentialsId: "${env.GIT_CRED_ID}"
                                bat """
                                    docker build -t ${env.IMAGE_REGISTRY}/backend:${branch}-${env.IMAGE_TAG} .
                                    docker push ${env.IMAGE_REGISTRY}/backend:${branch}-${env.IMAGE_TAG}
                                """
                            }
                        }
                    }
                }
            }
        }

        stage('Update Kubernetes Deployments') {
            steps {
                script {
                    def branches = env.BUILD_BRANCHES.split(',')
                    branches.each { branch ->
                        def feImage = "${env.IMAGE_REGISTRY}/frontend:${branch}-${env.IMAGE_TAG}"
                        def beImage = "${env.IMAGE_REGISTRY}/backend:${branch}-${env.IMAGE_TAG}"

                        bat "kubectl set image deployment/frontend-${branch} frontend=${feImage} --namespace=default"
                        bat "kubectl set image deployment/backend-${branch} backend=${beImage} --namespace=default"
                    }
                }
            }
        }
    }
}

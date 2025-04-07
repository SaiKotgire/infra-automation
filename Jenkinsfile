pipeline {
    agent any
    environment {
        FRONTEND_REPO = 'https://github.com/SaiKotgire/frontend.git' // Replace with your actual URL
        BACKEND_REPO = 'https://github.com/SaiKotgire/backend.git'
        IMAGE_REGISTRY = 'saikotgirep' // DockerHub username
        GIT_CRED_ID = 'github-https' // Set in Jenkins > Credentials
    }

    stages {
        stage('Determine Build Branches') {
            steps {
                script {
                    def today = new Date()
                    def day = today.format('d') as Integer
                    def builds = []

                    builds << 'main' // Always

                    if (day % 2 == 1) {
                        builds << 'dev' // Odd days
                    }

                    if (day % 2 == 0) {
                        builds << 'stage' // Every 2 days
                    }

                    env.BUILD_BRANCHES = builds.join(',')
                }
            }
        }

        stage('Clone and Build Docker Images') {
            steps {
                script {
                    def branches = env.BUILD_BRANCHES.split(',')
                    def timestamp = sh(script: "date +%Y%m%d-%H%M%S", returnStdout: true).trim()

                    branches.each { branch ->
                        // Clone and Build Frontend
                        dir("frontend-${branch}") {
                            git url: "${env.FRONTEND_REPO}", branch: branch, credentialsId: "${env.GIT_CRED_ID}"
                            sh "docker build -t ${IMAGE_REGISTRY}/frontend:${branch}-${timestamp} ."
                            sh "docker push ${IMAGE_REGISTRY}/frontend:${branch}-${timestamp}"
                        }

                        // Clone and Build Backend
                        dir("backend-${branch}") {
                            git url: "${env.BACKEND_REPO}", branch: branch, credentialsId: "${env.GIT_CRED_ID}"
                            sh "docker build -t ${IMAGE_REGISTRY}/backend:${branch}-${timestamp} ."
                            sh "docker push ${IMAGE_REGISTRY}/backend:${branch}-${timestamp}"
                        }
                    }

                    env.IMAGE_TAG = timestamp
                }
            }
        }

        stage('Update Kubernetes Deployments') {
            steps {
                script {
                    def branches = env.BUILD_BRANCHES.split(',')
                    branches.each { branch ->
                        def feImage = "${IMAGE_REGISTRY}/frontend:${branch}-${env.IMAGE_TAG}"
                        def beImage = "${IMAGE_REGISTRY}/backend:${branch}-${env.IMAGE_TAG}"

                        // Replace deployment names with your actual K8s deployment names
                        sh "kubectl set image deployment/frontend-${branch} frontend=${feImage}"
                        sh "kubectl set image deployment/backend-${branch} backend=${beImage}"
                    }
                }
            }
        }

        stage('Notify') {
            steps {
                mail to: 'your-email@example.com',
                     subject: "✅ Deployment Successful",
                     body: "Deployed frontend/backend for branches: ${env.BUILD_BRANCHES}"
            }
        }
    }
}

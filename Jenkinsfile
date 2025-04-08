pipeline {
    agent any

    environment {
        FRONTEND_REPO = 'https://github.com/SaiKotgire/frontend.git'
        BACKEND_REPO = 'https://github.com/SaiKotgire/backend.git'
        IMAGE_REGISTRY = 'saikotgirep'
        GIT_CRED_ID = 'github-https'
    }

    stages {
        stage('Determine Build Branches') {
            steps {
                script {
                    def dayOfWeek = new Date().format('EEEE', TimeZone.getTimeZone('Asia/Kolkata'))
                    def builds = ['main'] // Always build main

                    if (dayOfWeek == 'Monday') {
                        builds << 'testing'
                    }
                    if (dayOfWeek == 'Tuesday' || dayOfWeek == 'Thursday') {
                        builds << 'staging'
                    }

                    env.BUILD_BRANCHES = builds.join(',')
                    echo "Today is ${dayOfWeek}, building branches: ${env.BUILD_BRANCHES}"
                }
            }
        }

        stage('Clone and Build Docker Images') {
            steps {
                script {
                    def branches = env.BUILD_BRANCHES.split(',')

                    // Generate timestamp using PowerShell (for Windows agent)
                    def timestamp = powershell(script: '[DateTime]::Now.ToString("yyyyMMdd-HHmmss")', returnStdout: true).trim()
                    env.IMAGE_TAG = timestamp

                    withCredentials([usernamePassword(credentialsId: 'dockerhub', usernameVariable: 'DOCKER_USERNAME', passwordVariable: 'DOCKER_PASSWORD')]) {
                        bat "echo ${DOCKER_PASSWORD} | docker login -u ${DOCKER_USERNAME} --password-stdin"

                        branches.each { branch ->
                            // Frontend
                            dir("frontend-${branch}") {
                                git url: "${env.FRONTEND_REPO}", branch: branch, credentialsId: "${env.GIT_CRED_ID}"
                                bat "docker build -t ${IMAGE_REGISTRY}/frontend:${branch}-${env.IMAGE_TAG} ."
                                bat "docker push ${IMAGE_REGISTRY}/frontend:${branch}-${env.IMAGE_TAG}"
                            }

                            // Backend
                            dir("backend-${branch}") {
                                git url: "${env.BACKEND_REPO}", branch: branch, credentialsId: "${env.GIT_CRED_ID}"
                                bat "docker build -t ${IMAGE_REGISTRY}/backend:${branch}-${env.IMAGE_TAG} ."
                                bat "docker push ${IMAGE_REGISTRY}/backend:${branch}-${env.IMAGE_TAG}"
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
                        def feImage = "${IMAGE_REGISTRY}/frontend:${branch}-${env.IMAGE_TAG}"
                        def beImage = "${IMAGE_REGISTRY}/backend:${branch}-${env.IMAGE_TAG}"

                        bat "kubectl set image deployment/frontend-${branch} frontend=${feImage} --namespace=default"
                        bat "kubectl set image deployment/backend-${branch} backend=${beImage} --namespace=default"
                    }
                }
            }
        }

        stage('Notify') {
            steps {
                mail to: 'your-email@example.com',
                     subject: "✅ Deployment Successful - ${env.IMAGE_TAG}",
                     body: "Frontend and backend deployed for branches: ${env.BUILD_BRANCHES}"
            }
        }
    }
}

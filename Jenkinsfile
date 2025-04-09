pipeline {
    agent any

    environment {
        FRONTEND_REPO = 'https://github.com/SaiKotgire/frontend.git'
        BACKEND_REPO = 'https://github.com/SaiKotgire/backend.git'
        IMAGE_REGISTRY = 'saikotgirep'
        GIT_CRED_ID = 'github-https'
        // Add Docker Hub credentials ID that you've configured in Jenkins
        DOCKER_CRED_ID = 'dockerhub'
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
                        builds << 'stage'
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

                    // Generate timestamp in a cross-platform way
                    def timestamp = sh(script: 'date +"%Y%m%d-%H%M%S"', returnStdout: true).trim()
                    env.IMAGE_TAG = timestamp

                    branches.each { branch ->
                        // Frontend
                        dir("frontend-${branch}") {
                            git url: "${env.FRONTEND_REPO}", branch: branch, credentialsId: "${env.GIT_CRED_ID}"
                            
                            // Secure Docker build and push
                            withCredentials([usernamePassword(
                                credentialsId: env.DOCKER_CRED_ID,
                                usernameVariable: 'DOCKER_USERNAME',
                                passwordVariable: 'DOCKER_PASSWORD'
                            )]) {
                                sh """
                                    docker build -t ${env.IMAGE_REGISTRY}/frontend:${branch}-${env.IMAGE_TAG} .
                                    echo \$DOCKER_PASSWORD | docker login -u \$DOCKER_USERNAME --password-stdin
                                    docker push ${env.IMAGE_REGISTRY}/frontend:${branch}-${env.IMAGE_TAG}
                                """
                            }
                        }

                        // Backend
                        dir("backend-${branch}") {
                            git url: "${env.BACKEND_REPO}", branch: branch, credentialsId: "${env.GIT_CRED_ID}"
                            
                            withCredentials([usernamePassword(
                                credentialsId: env.DOCKER_CRED_ID,
                                usernameVariable: 'DOCKER_USERNAME',
                                passwordVariable: 'DOCKER_PASSWORD'
                            )]) {
                                sh """
                                    docker build -t ${env.IMAGE_REGISTRY}/backend:${branch}-${env.IMAGE_TAG} .
                                    echo \$DOCKER_PASSWORD | docker login -u \$DOCKER_USERNAME --password-stdin
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

                        sh "kubectl set image deployment/frontend-${branch} frontend=${feImage} --namespace=default"
                        sh "kubectl set image deployment/backend-${branch} backend=${beImage} --namespace=default"
                    }
                }
            }
        }

        stage('Notify') {
            steps {
                emailext(
                    to: 'your-email@example.com',
                    subject: "✅ Deployment Successful - ${env.IMAGE_TAG}",
                    body: """
                        <p>Frontend and backend deployed successfully for branches: ${env.BUILD_BRANCHES}</p>
                        <p>Deployment details:</p>
                        <ul>
                            <li>Timestamp: ${env.IMAGE_TAG}</li>
                            <li>Branches: ${env.BUILD_BRANCHES}</li>
                            <li>Registry: ${env.IMAGE_REGISTRY}</li>
                        </ul>
                    """,
                    mimeType: 'text/html'
                )
            }
        }
    }
}

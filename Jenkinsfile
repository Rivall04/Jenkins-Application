pipeline {
    agent any

    tools {
        nodejs 'NodeJS'
    }

    environment {
        DOCKERHUB_CREDENTIALS = credentials('dockerhub-credentials')
        FRONTEND_IMAGE = 'refalalhazmi/frontend-app'
        BACKEND_IMAGE  = 'refalalhazmi/backend-app'
        
        GIT_COMMIT_REV = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
        VERSION = "${BRANCH_NAME}-${GIT_COMMIT_REV}"
        
        SKIP_BUILD = 'false'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Check If Image Exists') {
            when {
                anyOf { branch 'production'; branch 'staging' }
            }
            steps {
                script {
                    echo "Checking Registry for version: ${VERSION}"
                    def frontendExists = sh(script: "docker manifest inspect ${FRONTEND_IMAGE}:${VERSION} > /dev/null 2>&1", returnStatus: true)
                    def backendExists  = sh(script: "docker manifest inspect ${BACKEND_IMAGE}:${VERSION} > /dev/null 2>&1", returnStatus: true)

                    if (frontendExists == 0 && backendExists == 0) {
                        echo " Image ${VERSION} already exists in Docker Hub. Skipping build/push."
                        SKIP_BUILD = 'true'
                    } else {
                        echo "Image not found. Starting build process..."
                        SKIP_BUILD = 'false'
                    }
                }
            }
        }

        stage('Install & Build') {
            when { expression { SKIP_BUILD == 'false' } }
            steps {
                sh """
                  cd frontend && npm install --no-audit --no-fund && npm run build
                  cd ../backend && npm install --no-audit --no-fund
                """
            }
        }

        stage('Build Docker Image') {
            when { expression { SKIP_BUILD == 'false' } }
            steps {
                sh """
                  docker build -t ${FRONTEND_IMAGE}:${VERSION} frontend
                  docker build -t ${BACKEND_IMAGE}:${VERSION} backend
                """
            }
        }

        stage('Security Scan') {
            when { expression { SKIP_BUILD == 'false' } }
            steps {
                sh """
                  docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
                    aquasec/trivy image --severity HIGH,CRITICAL ${FRONTEND_IMAGE}:${VERSION}
                """
            }
        }

        stage('Push to Docker Hub') {
            when {
                allOf {
                    expression { SKIP_BUILD == 'false' }
                    anyOf { branch 'production'; branch 'staging'; branch 'development' }
                }
            }
            steps {
                sh 'echo "${DOCKERHUB_CREDENTIALS_PSW}" | docker login -u "${DOCKERHUB_CREDENTIALS_USR}" --password-stdin'
                sh """
                  docker push ${FRONTEND_IMAGE}:${VERSION}
                  docker push ${BACKEND_IMAGE}:${VERSION}
                """
            }
        }
    }

    post {
        always {
            sh "docker rmi ${FRONTEND_IMAGE}:${VERSION} ${BACKEND_IMAGE}:${VERSION} || true"
            cleanWs()
        }
    }
}

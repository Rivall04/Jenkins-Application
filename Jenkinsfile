pipeline {
    agent any

    tools {
        nodejs 'NodeJS'
    }

    environment {
        DOCKERHUB_CREDENTIALS = credentials('dockerhub-credentials')
        
        PROD_REPO_FRONT    = 'refalalhazmi/prod-frontend'
        STAGING_REPO_FRONT = 'refalalhazmi/staging-frontend'
        
        PROD_REPO_BACK     = 'refalalhazmi/prod-backend'
        STAGING_REPO_BACK  = 'refalalhazmi/staging-backend'

        TARGET_IMAGE_FRONT = "${BRANCH_NAME == 'production' ? PROD_REPO_FRONT : STAGING_REPO_FRONT}"
        TARGET_IMAGE_BACK  = "${BRANCH_NAME == 'production' ? PROD_REPO_BACK : STAGING_REPO_BACK}"
        
        GIT_TAG = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
        SKIP_BUILD = 'false'
    }

    stages {
        stage('Check Registry') {
            when { anyOf { branch 'production'; branch 'staging' } }
            steps {
                script {
                    echo "Checking Registry: ${TARGET_IMAGE_FRONT} for tag: ${GIT_TAG}"
                    
                    def imageExists = sh(script: "docker manifest inspect ${TARGET_IMAGE_FRONT}:${GIT_TAG} > /dev/null 2>&1", returnStatus: true)
                    
                    if (imageExists == 0) {
                        echo "Version ${GIT_TAG} already exists in ${TARGET_IMAGE_FRONT}. Skipping build."
                        SKIP_BUILD = 'true'
                    } else {
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

        stage('Docker Build & Push') {
            when {
                allOf {
                    expression { SKIP_BUILD == 'false' }
                    anyOf { branch 'production'; branch 'staging' }
                }
            }
            steps {
                sh 'echo "${DOCKERHUB_CREDENTIALS_PSW}" | docker login -u "${DOCKERHUB_CREDENTIALS_USR}" --password-stdin'
                sh """
                  docker build -t ${TARGET_IMAGE_FRONT}:${GIT_TAG} -t ${TARGET_IMAGE_FRONT}:latest frontend
                  docker build -t ${TARGET_IMAGE_BACK}:${GIT_TAG} -t ${TARGET_IMAGE_BACK}:latest backend
                  
                  docker push ${TARGET_IMAGE_FRONT}:${GIT_TAG}
                  docker push ${TARGET_IMAGE_FRONT}:latest
                  docker push ${TARGET_IMAGE_BACK}:${GIT_TAG}
                  docker push ${TARGET_IMAGE_BACK}:latest
                """
            }
        }
    }

    post {
        always {
            sh "docker rmi ${TARGET_IMAGE_FRONT}:${GIT_TAG} ${TARGET_IMAGE_FRONT}:latest || true"
            sh "docker rmi ${TARGET_IMAGE_BACK}:${GIT_TAG} ${TARGET_IMAGE_BACK}:latest || true"
            cleanWs()
        }
    }
}

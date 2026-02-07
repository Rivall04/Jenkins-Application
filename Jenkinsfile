pipeline {
    agent any

    tools {
        nodejs 'NodeJS'
    }

    environment {
        DOCKERHUB_CREDENTIALS = credentials('dockerhub-credentials')
        
        // 1. تحديد أسماء المستودعات لكل بيئة
        PROD_REPO_FRONT    = 'refalalhazmi/prod-frontend'
        STAGING_REPO_FRONT = 'refalalhazmi/staging-frontend'
        
        PROD_REPO_BACK     = 'refalalhazmi/prod-backend'
        STAGING_REPO_BACK  = 'refalalhazmi/staging-backend'

        // 2. اختيار المستودع الهدف بناءً على اسم الفرع الحالي
        // إذا كان الفرع production اختار مستودع البرودكشن، غير ذلك اختار الـ staging
        TARGET_IMAGE_FRONT = "${BRANCH_NAME == 'production' ? PROD_REPO_FRONT : STAGING_REPO_FRONT}"
        TARGET_IMAGE_BACK  = "${BRANCH_NAME == 'production' ? PROD_REPO_BACK : STAGING_REPO_BACK}"
        
        // تاق فريد باستخدام كود الـ Git
        GIT_TAG = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
        SKIP_BUILD = 'false'
    }

    stages {
        stage('Check Registry') {
            when { anyOf { branch 'production'; branch 'staging' } }
            steps {
                script {
                    echo "Checking Registry: ${TARGET_IMAGE_FRONT} for tag: ${GIT_TAG}"
                    
                    // فحص إذا كان هذا التاق موجود في المستودع المخصص لهذه البيئة
                    def imageExists = sh(script: "docker manifest inspect ${TARGET_IMAGE_FRONT}:${GIT_TAG} > /dev/null 2>&1", returnStatus: true)
                    
                    if (imageExists == 0) {
                        echo "✅ النسخة موجودة مسبقاً في المستودع. سيتم تخطي البناء."
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
                // استخدام التحسينات التي تكلمنا عنها سابقاً لتوفير الرام
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
                  # بناء ورفع الصور للمستودع المختار (إما Prod أو Staging)
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
            // تنظيف الجهاز من الصور المحلية بعد الرفع
            sh "docker rmi ${TARGET_IMAGE_FRONT}:${GIT_TAG} ${TARGET_IMAGE_FRONT}:latest || true"
            sh "docker rmi ${TARGET_IMAGE_BACK}:${GIT_TAG} ${TARGET_IMAGE_BACK}:latest || true"
            cleanWs()
        }
    }
}

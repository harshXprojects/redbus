pipeline {
    agent any

    environment {
        BACKEND_IMAGE  = "harshxprojects/redbus-backend"
        FRONTEND_IMAGE = "harshxprojects/redbus-frontend"
        // Credentials stored in Jenkins → Manage Credentials
        DOCKERHUB_CREDS = credentials('dockerhub-credentials')   // Username/Password kind
        GIT_CREDS       = credentials('github-token')            // Secret text kind
    }

    tools {
        nodejs 'NodeJS-20'   // configure this name in Jenkins → Global Tool Configuration
    }

    options {
        timestamps()
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }

    stages {

        
        stage('Checkout') {
        
            steps {
                checkout scm
                script {
                    env.GIT_SHA_SHORT = sh(
                        script: "git rev-parse --short HEAD",
                        returnStdout: true
                    ).trim()
                    echo "Building commit: ${env.GIT_SHA_SHORT}"
                }
            }
        }

        
        stage('Install Dependencies') {
        
            parallel {
                stage('Backend: Install') {
                    steps {
                        dir('back-end-redbus') {
                            sh 'npm ci'
                        }
                    }
                }
                stage('Frontend: Install') {
                    steps {
                        dir('front-end-redbus') {
                            sh 'npm ci'
                        }
                    }
                }
            }
        }

        
        stage('Test') {
        
            parallel {
                stage('Backend: Test') {
                    steps {
                        dir('back-end-redbus') {
                            sh 'npm test --if-present'
                        }
                    }
                }
                stage('Frontend: Test') {
                    steps {
                        dir('front-end-redbus') {
                            sh 'CI=true npm test --if-present -- --watchAll=false'
                        }
                    }
                }
            }
        }

        
        stage('Docker Build') {
        
            when {
                anyOf {
                    branch 'master'
                    branch 'main'
                }
            }
            parallel {
                stage('Backend: Build Image') {
                    steps {
                        dir('back-end-redbus') {
                            sh """
                                docker build \
                                    -t ${BACKEND_IMAGE}:${GIT_SHA_SHORT} \
                                    -t ${BACKEND_IMAGE}:latest \
                                    .
                            """
                        }
                    }
                }
                stage('Frontend: Build Image') {
                    steps {
                        dir('front-end-redbus') {
                            sh """
                                docker build \
                                    -t ${FRONTEND_IMAGE}:${GIT_SHA_SHORT} \
                                    -t ${FRONTEND_IMAGE}:latest \
                                    .
                            """
                        }
                    }
                }
            }
        }

       
        stage('Push to Docker Hub') {
        
            when {
                anyOf {
                    branch 'master'
                    branch 'main'
                }
            }
            steps {
                sh "echo ${DOCKERHUB_CREDS_PSW} | docker login -u ${DOCKERHUB_CREDS_USR} --password-stdin"
                sh """
                    docker push ${BACKEND_IMAGE}:${GIT_SHA_SHORT}
                    docker push ${BACKEND_IMAGE}:latest
                    docker push ${FRONTEND_IMAGE}:${GIT_SHA_SHORT}
                    docker push ${FRONTEND_IMAGE}:latest
                """
            }
        }

        
        stage('Validate K8s Manifests') {
        
            steps {
                sh """
                    kubectl apply --dry-run=client -f k8s/namespace.yaml
                    kubectl apply --dry-run=client -f k8s/secret.yaml
                    kubectl apply --dry-run=client -f k8s/configmap.yaml
                    kubectl apply --dry-run=client -f k8s/mongodb/
                    kubectl apply --dry-run=client -f k8s/backend/
                    kubectl apply --dry-run=client -f k8s/frontend/
                    kubectl apply --dry-run=client -f k8s/ingress.yaml
                """
            }
        }

        
        stage('Update K8s Manifests (GitOps)') {
        
            when {
                anyOf {
                    branch 'master'
                    branch 'main'
                }
            }
            steps {
                sh """
                    sed -i 's|${BACKEND_IMAGE}:.*|${BACKEND_IMAGE}:${GIT_SHA_SHORT}|g' k8s/backend/deployment.yaml
                    sed -i 's|${FRONTEND_IMAGE}:.*|${FRONTEND_IMAGE}:${GIT_SHA_SHORT}|g' k8s/frontend/deployment.yaml
                """

                sh """
                    git config user.name  "jenkins-bot"
                    git config user.email "jenkins-bot@redbus.ci"

                    git add k8s/backend/deployment.yaml k8s/frontend/deployment.yaml

                    # Only commit if there are actual changes
                    git diff --staged --quiet || \
                        git commit -m "ci: bump images to ${GIT_SHA_SHORT} [skip ci]"

                    git push https://${GIT_CREDS}@github.com/harshXprojects/redbus.git HEAD:master
                """
            }
        }

        
        stage('ArgoCD Sync') {
        
            when {
                anyOf {
                    branch 'master'
                    branch 'main'
                }
            }
            steps {
                withCredentials([string(credentialsId: 'argocd-token', variable: 'ARGOCD_TOKEN')]) {
                    sh """
                        argocd app sync redbus \
                            --auth-token ${ARGOCD_TOKEN} \
                            --server <your-argocd-server>:443 \
                            --grpc-web \
                            --timeout 120
                    """
                }
            }
        }

    } 

    post {
        always {
            sh 'docker logout || true'
            cleanWs()
        }
        success {
            echo "✅ Pipeline passed — image tag: ${env.GIT_SHA_SHORT}"
        }
        failure {
            echo "❌ Pipeline failed on branch: ${env.BRANCH_NAME}"
        }
    }
}

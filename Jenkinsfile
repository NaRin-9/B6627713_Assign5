pipeline {
    agent any

    tools {
        dockerTool 'docker-cli'
    }

    triggers {
        githubPush()
    }

    environment {
        DOCKER_USER = 'narin7'
        APP_NAME    = 'my-nginx-web'
        IMAGE_TAG   = "${BUILD_NUMBER}"
        DOCKER_HOST = 'tcp://host.docker.internal:2375'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build & Push Docker Image') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-credentials', passwordVariable: 'DOCKER_PASS', usernameVariable: 'DOCKER_USER_ID')]) {
                    script {
                        sh '''
                        if ! command -v docker &> /dev/null; then
                            echo "Downloading updated Docker CLI..."
                            curl -fsSL https://download.docker.com/linux/static/stable/x86_64/docker-26.0.0.tgz -o docker.tgz
                            tar xzvf docker.tgz
                            export PATH=$PATH:$(pwd)/docker
                        fi
                        
                        echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER_ID" --password-stdin
                        
                        docker build -t ${DOCKER_USER}/${APP_NAME}:${IMAGE_TAG} -t ${DOCKER_USER}/${APP_NAME}:latest .
                        
                        docker push ${DOCKER_USER}/${APP_NAME}:${IMAGE_TAG}
                        docker push ${DOCKER_USER}/${APP_NAME}:latest
                        '''
                    }
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                // เปลี่ยนมาใช้แบบ file และเรียกชื่อ ID ใหม่
                withCredentials([file(credentialsId: 'kubeconfig-file', variable: 'KUBECONFIG_PATH')]) {
                    script {
                        sh '''
                        if [ ! -f "./kubectl" ]; then
                            echo "Downloading kubectl..."
                            curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
                            chmod +x ./kubectl
                        fi
                        
                        # ชี้ที่อยู่ไฟล์ไปที่ที่ Jenkins เตรียมไว้ให้เลย ไม่ต้อง echo สร้างไฟล์เองแล้ว
                        export KUBECONFIG=$KUBECONFIG_PATH
                        
                        ./kubectl apply -f k8s/deployment.yaml
                        ./kubectl apply -f k8s/service.yaml
                        ./kubectl apply -f k8s/ingress.yaml
                        
                        ./kubectl set image deployment/nginx-deployment nginx-container=${DOCKER_USER}/${APP_NAME}:${IMAGE_TAG}
                        '''
                    }
                }
            }
        }

        stage('Verify Deployment') {
            steps {
                withCredentials([file(credentialsId: 'kubeconfig-file', variable: 'KUBECONFIG_PATH')]) {
                    script {
                        sh '''
                        export KUBECONFIG=$KUBECONFIG_PATH
                        
                        ./kubectl rollout status deployment/nginx-deployment --timeout=120s
                        ./kubectl get pods
                        ./kubectl get svc
                        ./kubectl get ingress
                        '''
                    }
                }
            }
        }
    }

    post {
        success {
            echo "Deployment successful! Access at http://7713.jenkins"
        }
        failure {
            echo "Deployment failed! Check logs for details."
        }
    }
}

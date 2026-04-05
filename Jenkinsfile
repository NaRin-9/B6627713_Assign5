pipeline {
    agent any

    tools {
        dockerTool 'docker-cli'
    }

    triggers {
        githubPush()
    }

    environment {
        // ตั้งค่าตัวแปรต่างๆ ให้เรียบร้อย
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
                withCredentials([string(credentialsId: 'kubeconfig-local', variable: 'KUBECONFIG_TXT')]) {
                    script {
                        sh '''
                        if [ ! -f "./kubectl" ]; then
                            echo "Downloading kubectl..."
                            curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
                            chmod +x ./kubectl
                        fi
                        
                        echo "$KUBECONFIG_TXT" > kubeconfig_tmp
                        export KUBECONFIG=$(pwd)/kubeconfig_tmp
                        
                        # รันไฟล์ YAML (ถ้าโฟลเดอร์ใน GitHub ของคุณชื่ออื่นที่ไม่ใช่ jenkins/ ให้แก้ตรงนี้ด้วยนะครับ)
                        ./kubectl apply -f jenkins/deployment.yaml
                        ./kubectl apply -f jenkins/service.yaml
                        ./kubectl apply -f jenkins/ingress.yaml
                        
                        # อัปเดตเวอร์ชัน Image
                        ./kubectl set image deployment/nginx-deployment nginx-container=${DOCKER_USER}/${APP_NAME}:${IMAGE_TAG}
                        '''
                    }
                }
            }
        }

        stage('Verify Deployment') {
            steps {
                withCredentials([string(credentialsId: 'kubeconfig-local', variable: 'KUBECONFIG_TXT')]) {
                    script {
                        sh '''
                        export KUBECONFIG=$(pwd)/kubeconfig_tmp
                        
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
        always {
            sh "rm -f kubeconfig_tmp || true"
        }
        success {
            echo "Deployment successful! Access at http://7713.jenkins"
        }
        failure {
            echo "Deployment failed! Check logs for details."
        }
    }
}

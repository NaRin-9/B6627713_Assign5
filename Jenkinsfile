pipeline {
    // ใช้ agent any เพื่อให้รันบนเครื่องหลักได้เลย ไม่ต้องรอคิว k8s-agent-1
    agent any

    // เรียกใช้ Docker Tool ที่เราไปสร้างไว้ใน Manage Jenkins > Tools
    tools {
        dockerTool 'docker-cli'
    }

    triggers {
        githubPush()
    }

    environment {
        // ⚠️ สำคัญมาก: เปลี่ยนตรงนี้เป็น Username ของ Docker Hub ของคุณ
        DOCKER_USER = 'narin7'
        
        APP_NAME    = 'my-nginx-web'
        IMAGE_TAG   = "${BUILD_NUMBER}"
        
        // ชี้เป้าหมายให้ Jenkins วิ่งทะลุไปหา Docker Desktop บน Windows
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
                script {
                    // 1. Build Image โดยใส่ Username นำหน้า เพื่อเตรียมส่งขึ้น Docker Hub
                    sh "docker build -t ${DOCKER_USER}/${APP_NAME}:${IMAGE_TAG} -t ${DOCKER_USER}/${APP_NAME}:latest ."
                    
                    // 2. Push Image ขึ้นไปเก็บบน Docker Hub (Kind Cluster จะได้ดึงไปติดตั้งได้)
                    // (ถ้าตอนรันติดปัญหา Unauthenticated ให้ไปรันคำสั่ง docker login ใน PowerShell ที่เครื่อง Windows ก่อน 1 ครั้งครับ)
                    sh "docker push ${DOCKER_USER}/${APP_NAME}:${IMAGE_TAG}"
                    sh "docker push ${DOCKER_USER}/${APP_NAME}:latest"
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                // เรียกใช้ กุญแจ Kubeconfig ที่เราเซฟไว้ใน Credentials
                withCredentials([string(credentialsId: 'kubeconfig-local', variable: 'KUBECONFIG_TXT')]) {
                    script {
                        sh '''
                        # สร้างไฟล์ config ชั่วคราวให้ kubectl ใช้
                        echo "$KUBECONFIG_TXT" > kubeconfig_tmp
                        export KUBECONFIG=$(pwd)/kubeconfig_tmp
                        
                        # ⚠️ ตรวจสอบ Path: ถ้าไฟล์ YAML ของคุณอยู่ในโฟลเดอร์ jenkins ให้เปลี่ยนจาก k8s/ เป็น jenkins/ นะครับ
                        kubectl apply -f k8s/deployment.yaml
                        kubectl apply -f k8s/service.yaml
                        kubectl apply -f k8s/ingress.yaml
                        
                        # สั่งเปลี่ยน Image ใน Deployment ให้เป็นเวอร์ชันใหม่ล่าสุดที่เพิ่ง Build
                        kubectl set image deployment/nginx-deployment nginx-container=${DOCKER_USER}/${APP_NAME}:${IMAGE_TAG}
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
                        
                        kubectl rollout status deployment/nginx-deployment --timeout=120s
                        kubectl get pods -l app=my-nginx
                        kubectl get svc nginx-service
                        kubectl get ingress nginx-ingress
                        '''
                    }
                }
            }
        }
    }

    post {
        always {
            // ทำความสะอาด ลบไฟล์กุญแจทิ้งทุกครั้งที่รันเสร็จเพื่อความปลอดภัย
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

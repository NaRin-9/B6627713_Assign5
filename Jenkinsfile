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
                // เพิ่มบล็อก withCredentials เพื่อดึงรหัสผ่านที่เราตั้งไว้มาใช้งาน
                withCredentials([usernamePassword(credentialsId: 'dockerhub-credentials', passwordVariable: 'DOCKER_PASS', usernameVariable: 'DOCKER_USER_ID')]) {
                    script {
                        sh '''
                        if ! command -v docker &> /dev/null; then
                            echo "Downloading updated Docker CLI..."
                            curl -fsSL https://download.docker.com/linux/static/stable/x86_64/docker-26.0.0.tgz -o docker.tgz
                            tar xzvf docker.tgz
                            export PATH=$PATH:$(pwd)/docker
                        fi
                        
                        # 1. ล็อกอินเข้า Docker Hub ก่อน
                        echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER_ID" --password-stdin
                        
                        # 2. Build Image
                        docker build -t narin7/my-nginx-web:${BUILD_NUMBER} -t narin7/my-nginx-web:latest .
                        
                        # 3. Push Image (คราวนี้จะผ่านแล้วเพราะล็อกอินแล้ว!)
                        docker push narin7/my-nginx-web:${BUILD_NUMBER}
                        docker push narin7/my-nginx-web:latest
                        '''
                    }
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

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
                withCredentials([string(credentialsId: 'kubeconfig-local', variable: 'KUBECONFIG_TXT')]) {
                    script {
                        sh '''
                        # 1. โหลดโปรแกรม kubectl มาไว้ในโปรเจกต์ (ถ้ายังไม่มี)
                        if [ ! -f "./kubectl" ]; then
                            echo "Downloading kubectl..."
                            curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
                            chmod +x ./kubectl
                        fi
                        
                        # 2. เตรียมไฟล์กุญแจ Kubeconfig
                        echo "$KUBECONFIG_TXT" > kubeconfig_tmp
                        export KUBECONFIG=$(pwd)/kubeconfig_tmp
                        
                        # 3. เริ่ม Deploy (สังเกตว่าเราจะใช้ ./kubectl แทน kubectl เฉยๆ)
                        # ⚠️ สำคัญ: ถ้าไฟล์ของคุณอยู่ในโฟลเดอร์ jenkins ให้เปลี่ยนคำว่า k8s เป็น jenkins ด้วยนะครับ
                        ./kubectl apply -f jenkins/deployment.yaml
                        ./kubectl apply -f jenkins/service.yaml
                        ./kubectl apply -f jenkins/ingress.yaml
                        
                        # สั่งอัปเดต Image เป็นเวอร์ชันล่าสุดที่เพิ่ง Build
                        ./kubectl set image deployment/nginx-deployment nginx-container=narin7/my-nginx-web:${BUILD_NUMBER}
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
                        
                        # รอเช็คสถานะว่า Pods รันขึ้นมาครบไหม (ใช้ ./kubectl เช่นกัน)
                        ./kubectl rollout status deployment/nginx-deployment --timeout=120s
                        
                        # ดูสถานะต่างๆ
                        ./kubectl get pods
                        ./kubectl get svc
                        ./kubectl get ingress
                        '''
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

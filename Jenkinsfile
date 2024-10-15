pipeline {
    agent any

    environment{
        cred = credentials('aws-key') // AWS access key için tanımlı credential
        dockerhub_cred = credentials('docker-cred') // Docker Hub için tanımlı credential
        DOCKER_IMAGE = "srhnyldz/flask-monitoring"
        DOCKER_TAG = "$BUILD_NUMBER"
        SONARQUBE_URL = 'http://localhost:9000/'
        SONAR_TOKEN = credentials('SONAR_TOKEN')
    }

    stages {
        stage('Git Cloning') {
            steps {
                echo 'Cloning git repo'
                git url: 'https://github.com/srhnyldz/flask-monitoring.git', branch: 'main'
            }
        }
       stage("Docker Build & Push"){ // Docker image build ve push aşaması
            steps{
                script{
                   withDockerRegistry(credentialsId: 'docker-cred', toolName: 'docker') {                        
                        sh "docker build -t ${DOCKER_IMAGE}:${DOCKER_TAG} ."
                        sh "docker push ${DOCKER_IMAGE}:${DOCKER_TAG}"
                    }
                }
            }
        }
        stage("Update Kubernetes Manifest"){ // Kubernetes manifest dosyasını güncelleme
            steps{
                sh "sed -i 's|srhnyldz/flask-monitoring:latest|${DOCKER_IMAGE}:${DOCKER_TAG}|' ./deployment.yaml"
            }
        }
        stage("TRIVY"){
            steps{
                 sh "trivy image --scanners vuln ${DOCKER_IMAGE}:${DOCKER_TAG}"
            }
        }
        stage('Integrate Kubernetes with Jenkins') {
            steps {
                withCredentials([string(credentialsId: 'SECRET_TOKEN', variable: 'KUBE_TOKEN')]) {
                    sh '''
                    # Kubectl indirme ve çalışma izinlerini verme
                    curl -LO "https://storage.googleapis.com/kubernetes-release/release/v1.20.5/bin/linux/amd64/kubectl"
                    chmod u+x ./kubectl

                    # Kubernetes config ayarlarını yapma (sunucu IP'si, token ve güvenlik ayarları)
                    export KUBECONFIG=$(mktemp)
                    ./kubectl config set-cluster kubernetes --server=https://172.31.17.240:6443 --insecure-skip-tls-verify=true
                    ./kubectl config set-credentials jenkins --token=${KUBE_TOKEN}
                    ./kubectl config set-context default --cluster=jenkins-cluster --user=jenkins --namespace=default
                    ./kubectl config use-context default

                    # Kubernetes ile iletişim test etme
                    ./kubectl get nodes

                    # Deployment ve Service dosyalarını Kubernetes'e uygulama
                    ./kubectl apply -f service.yaml
                    ./kubectl apply -f deployment.yaml
                    '''
                }
            }
        }
    }
}

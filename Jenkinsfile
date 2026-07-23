pipeline {
    agent any
    environment {
        AWS_ACCOUNT_ID = '566540245122' 
        AWS_DEFAULT_REGION = 'ap-south-1' 
        IMAGE_REPO_NAME = 'prime-clone'
        IMAGE_TAG = "${BUILD_NUMBER}"
        GITHUB_CRED_ID = 'github-token'
    }
    stages {
        stage('Code Checkout') {
            steps {
                checkout scm
            }
        }
        
        stage('Terraform Provision Infrastructure') {
            steps {
                dir('terraform') {
                    // Initializes and provisions the VPC and single c7i-flex.large node automatically
                    sh 'terraform init'
                    sh 'terraform apply --auto-approve'
                }
            }
        }

        stage('Establish Cluster Access') {
            steps {
                // Connects your tools remote control context directly to the EKS cluster
                sh "aws eks update-kubeconfig --region $us-east-1 --name prime-poc-cluster"
            }
        }

        stage('NPM Install & Build') {
            steps {
                sh 'npm install'
                sh 'npm run build'
            }
        }

        stage('Docker Build Container') {
            steps {
                sh "docker build -t ${IMAGE_REPO_NAME}:${IMAGE_TAG} ."
                sh "docker images "
            }
        }
        
        stage('SonarQube Scan') {
            steps {
                script {
                    def scannerHome = tool 'sonar-scanner'
        
                    withCredentials([string(credentialsId: 'sonar-token', variable: 'SONAR_TOKEN')]) {
                        sh """
                        ${scannerHome}/bin/sonar-scanner \
                        -Dsonar.projectKey=prime-clone \
                        -Dsonar.sources=. \
                        -Dsonar.host.url=http://54.162.144.178:9000 \
                        -Dsonar.login=${SONAR_TOKEN}
                        """
                    }
                }
            }
        }

        stage('Trivy Scan') {
            steps {
                sh """
                docker run --rm \
                -v /var/run/docker.sock:/var/run/docker.sock \
                aquasec/trivy:latest image \
                --exit-code 0 \
                --severity HIGH,CRITICAL \
                --format table \
                ${IMAGE_REPO_NAME}:${IMAGE_TAG}
                """
            }
        }

        stage('Push Image to AWS ECR') {
            steps {
                sh """
                aws ecr get-login-password --region ${AWS_DEFAULT_REGION} | docker login --username AWS --password-stdin ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com
        
                docker tag ${IMAGE_REPO_NAME}:${IMAGE_TAG} ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com/${IMAGE_REPO_NAME}:latest
        
                docker tag ${IMAGE_REPO_NAME}:${IMAGE_TAG} ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com/${IMAGE_REPO_NAME}:${IMAGE_TAG}
        
                docker push ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com/${IMAGE_REPO_NAME}:latest
        
                docker push ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com/${IMAGE_REPO_NAME}:${IMAGE_TAG}
                """
            }
        }

        stage('Update Git Manifest For GitOps') {
            steps {
                withCredentials([usernamePassword(credentialsId: "${GITHUB_CRED_ID}", passwordVariable: 'GIT_PASSWORD', usernameVariable: 'GIT_USERNAME')]) {
                    sh """
                    # FIXED: Corrected domain format mapping using a forward slash and dollar variable sign prefix
                    sed -i "s|image: .*|image: ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com/${IMAGE_REPO_NAME}:${IMAGE_TAG}|g" k8s/deployment.yaml
                    
                    # 2. Configure Git operational profiles
                    git config user.email "jenkins@devsecops.poc"
                    git config user.name "Jenkins CI Engine"
                    
                    # 3. Stage and commit configuration adjustments
                    git add k8s/deployment.yaml
                    git commit -m "Automated build update: image tag v${IMAGE_TAG} [skip ci]" || true
                    
                    # 4. Clean, properly structured Git authentication routing string
                    git remote set-url origin "https://\${GIT_USERNAME}:\${GIT_PASSWORD}@github.com/\${GIT_USERNAME}/POC-6.git"
                    
                    # 5. Push deployment manifest changes straight up to GitHub main branch
                    git push origin HEAD:main
                    """
                }
            }
        }

        stage('Deploy GitOps & Helm Monitoring') {
            steps {
                sh '''
                # 1. Grant EKS nodes permission to pull images from ECR
                NODE_ROLE=$(aws iam list-roles --query "Roles[?contains(RoleName, 'monitoring_node')].RoleName" --output text)
        
                aws iam attach-role-policy \
                  --role-name $NODE_ROLE \
                  --policy-arn arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly || true
        
                # 2. Create ArgoCD namespace
                kubectl create namespace argocd || true
        
                # 3. Install ArgoCD using server-side apply
                kubectl apply --server-side -n argocd \
                  -f https://raw.githubusercontent.com/argoproj/argo-cd/v2.11.7/manifests/install.yaml || true
        
                # 4. Wait for ArgoCD resources to initialize completely
                kubectl wait --for=condition=available deployment/argocd-server \
                  -n argocd --timeout=300s || true
        
                # 5. Expose ArgoCD UI via NodePort
                kubectl patch svc argocd-server \
                  -n argocd \
                  -p '{"spec":{"type":"NodePort"}}' || true
        
                # 6. Deploy GitOps application
                kubectl apply -f k8s/argocd-app.yaml || true
        
                # 7. Add Prometheus Helm repo
                helm repo add prometheus-community \
                  https://prometheus-community.github.io/helm-charts || true
                helm repo update
        
                # 8. Create monitoring namespace
                kubectl create namespace monitoring || true
        
                # 9. Install Prometheus + Grafana inside the cluster node
                # FIXED: Added explicit nodePort exposure for the Prometheus Expression Browser service
                helm upgrade --install kube-stack \
                  prometheus-community/kube-prometheus-stack \
                  --namespace monitoring \
                  --set prometheus.prometheusSpec.resources.requests.memory=400Mi \
                  --set prometheus.prometheusSpec.resources.limits.memory=1200Mi \
                  --set grafana.service.type=NodePort \
                  --set prometheus.service.type=NodePort
        
                # 10. Verification logs
                kubectl get pods -n argocd
                kubectl get pods -n monitoring
                kubectl get pods -n default
                kubectl get deployment 
                kubectl get pods
                '''
            }
        }
        
        stage('Display Live Entry Details') {
            steps {
                sh '''
                echo "=========================================================="
                echo "🚀 DEVSECOPS POC LIFECYCLE CONNECTIONS 🚀"
                echo "=========================================================="
        
                # 1. Direct query to AWS EC2 API using EKS cluster tags to isolate the node's public IP
                NODE_IP=$(aws ec2 describe-instances \
                  --filters "Name=instance-state-name,Values=running" \
                            "Name=tag:kubernetes.io/cluster/prime-poc-cluster,Values=owned" \
                  --query "Reservations[*].Instances[*].PublicIpAddress" \
                  --output text | head -n1)
                
                # Fallback check if the node is entirely private without an elastic public IP mapping
                if [ -z "$NODE_IP" ] || [ "$NODE_IP" == "None" ]; then
                    echo "Public IP not found via cluster tags. Searching by instance type..."
                    NODE_IP=$(aws ec2 describe-instances \
                      --filters "Name=instance-state-name,Values=running" \
                                "Name=instance-type,Values=c7i-flex.large" \
                      --query "Reservations[*].Instances[*].PublicIpAddress" \
                      --output text | head -n1)
                fi
        
                # 2. Extract service NodePorts natively from inside the EKS cluster configurations
                ARGOCD_PORT=$(kubectl get svc argocd-server -n argocd -o jsonpath="{.spec.ports[0].nodePort}" 2>/dev/null || echo "N/A")
                GRAFANA_PORT=$(kubectl get svc kube-stack-grafana -n monitoring -o jsonpath="{.spec.ports[0].nodePort}" 2>/dev/null || echo "N/A")
                PROMETHEUS_PORT=$(kubectl get svc kube-stack-prometheus -n monitoring -o jsonpath="{.spec.ports[0].nodePort}" 2>/dev/null || echo "N/A")
                
                # 3. Securely decode service authentication secrets
                ARGOCD_PASS=$(kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" 2>/dev/null | base64 --decode || echo "N/A")
                GRAFANA_PASS=$(kubectl get secret -n monitoring kube-stack-grafana -o jsonpath="{.data.admin-password}" 2>/dev/null | base64 --decode || echo "N/A")
        
                # 4. Print clean, click-ready browser navigation routes, usernames, and passwords
                echo "🎬 Application URL : http://${NODE_IP}:32080"
                echo "   👉 Authentication: None (Public Application Endpoint)"
                echo "----------------------------------------------------------"
                echo "🐙 ArgoCD URL      : http://${NODE_IP}:${ARGOCD_PORT}"
                echo "   👤 User Name     : admin"
                # FIXED: Resolved broken double quotes structure causing compilation block failures
                echo "   🔑 Password      : ${ARGOCD_PASS}"
                echo "----------------------------------------------------------"
                echo "🔥 Prometheus URL  : http://${NODE_IP}:${PROMETHEUS_PORT}"
                echo "   👉 Authentication: None (Open Metrics Dashboard)"
                echo "----------------------------------------------------------"
                echo "📊 Grafana URL     : http://${NODE_IP}:${GRAFANA_PORT}"
                echo "   👤 User Name     : admin"
                echo "   🔑 Password      : ${GRAFANA_PASS}"
                echo "=========================================================="
                '''
            }
        }
    }
}

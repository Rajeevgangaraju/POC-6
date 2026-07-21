pipeline {
    agent any
    environment {
        AWS_ACCOUNT_ID = '567017110325' 
        AWS_DEFAULT_REGION = 'us-east-1' 
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
                sh "aws eks update-kubeconfig --region ${AWS_DEFAULT_REGION} --name prime-poc-cluster"
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
                    # 1. Update the Kubernetes manifest image reference using explicit variables
                    sed -i "s|image: .*|image: ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}://{IMAGE_REPO_NAME}:${IMAGE_TAG}|g" k8s/deployment.yaml
                    
                    # 2. Configure Git operational profiles
                    git config user.email "jenkins@devsecops.poc"
                    git config user.name "Jenkins CI Engine"
                    
                    # 3. Stage and commit configuration adjustments
                    git add k8s/deployment.yaml
                    git commit -m "Automated build update: image tag v${IMAGE_TAG} [skip ci]" || true
                    
                    # 4. FIXED: Clean, properly structured Git authentication routing string
                    git remote set-url origin "https://\${GIT_USERNAME}:\${GIT_PASSWORD}@github.com/\${GIT_USERNAME}/POC-6.git"
                    
                    # 5. Push deployment manifest changes straight up to GitHub main branch
                    git push origin HEAD:main
                    """
                }
            }
        }


        stage('Deploy GitOps & Helm Monitoring') {
            steps {
                sh """
                # 1. Provide EKS cluster node permissions to read container images from ECR
                NODE_ROLE=\$(aws iam list-roles --query "Roles[?contains(RoleName, 'monitoring_node')].RoleName" --output text)
                aws iam attach-role-policy --role-name \$NODE_ROLE --policy-arn arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly || true

                # 2. Deploy ArgoCD Engine Infrastructure Natively
                kubectl create namespace argocd || true
                
                # FIXED: Added the complete verified raw file pathway manifest endpoint
                kubectl apply -n argocd -f https://githubusercontent.com
                
                kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "NodePort"}}'

                # 3. Trigger Application Synchronization manifest loop
                kubectl apply -f k8s/argocd-app.yaml

                # 4. Provision Prometheus + Grafana cluster metrics stack via Helm
                helm repo add prometheus-community https://github.io || true
                helm repo update
                kubectl create namespace monitoring || true
                
                # Limits and controls memory allocations to fit exactly inside single-node computing parameters
                helm upgrade --install kube-stack prometheus-community/kube-prometheus-stack \
                  --namespace monitoring \
                  --set prometheus.prometheusSpec.resources.requests.memory=400Mi \
                  --set prometheus.prometheusSpec.resources.limits.memory=1200Mi \
                  --set grafana.service.type=NodePort
                """
            }
        }


        stage('Display Live Entry Details') {
            steps {
                sh """
                echo "=========================================================="
                echo "🚀 DEVSECOPS POC LIFECYCLE CONNECTIONS 🚀"
                echo "=========================================================="
                NODE_IP=\$(kubectl get nodes -o wide | awk 'NR==2 {print \$7}')
                ARGOCD_PORT=\$(kubectl get svc -n argocd argocd-server -o jsonpath='{.spec.ports[?(@.name=="http")].nodePort}')
                GRAFANA_PORT=\$(kubectl get svc -n monitoring kube-stack-grafana -o jsonpath='{.spec.ports[?(@.name=="http")].nodePort}')
                GRAFANA_PASS=\$(kubectl get secret -n monitoring kube-stack-grafana -o jsonpath='{.data.admin-password}' | base64 --decode)
                
                echo "🎬 APPLICATION MAIN PAGE URL: http://\${NODE_IP}:32080"
                echo "🐙 ARGOCD CONTROLLER BOARD:  http://\${NODE_IP}:\${ARGOCD_PORT}"
                echo "📊 GRAFANA CLUSTER METRICS:   http://\${NODE_IP}:\${GRAFANA_PORT}"
                echo "🔑 DEFAULT GRAFANA PASSWORD:  \${GRAFANA_PASS} (User: admin)"
                echo "=========================================================="
                """
            }
        }
    }
}

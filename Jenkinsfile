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
                    sh 'terraform init'
                    sh 'terraform apply --auto-approve'
                }
            }
        }

        stage('Establish Cluster Access') {
            steps {
                sh '''
                aws eks update-kubeconfig \
                --region ap-south-1 \
                --name prime-poc-cluster

                kubectl get nodes
                '''
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
                sh "docker images"
            }
        }

        stage('SonarQube Scan') {
            steps {
                script {
                    def scannerHome = tool 'sonar-scanner'

                    withCredentials([
                        string(
                            credentialsId: 'sonar-token',
                            variable: 'SONAR_TOKEN'
                        )
                    ]) {
                        sh """
                        ${scannerHome}/bin/sonar-scanner \
                        -Dsonar.projectKey=prime-clone \
                        -Dsonar.sources=. \
                        -Dsonar.host.url=http://13.203.42.55:9000 \
                        -Dsonar.token=$SONAR_TOKEN \
                        -Dsonar.exclusions=**/.terraform/**,**/node_modules/**,**/dist/**,**/build/**,**/.git/**
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

        stage('Push Image To AWS ECR') {
            steps {
                sh '''
                aws ecr get-login-password --region ap-south-1 | \
                docker login \
                --username AWS \
                --password-stdin \
                566540245122.dkr.ecr.ap-south-1.amazonaws.com

                docker tag prime-clone:${BUILD_NUMBER} \
                566540245122.dkr.ecr.ap-south-1.amazonaws.com/prime-clone:latest

                docker tag prime-clone:${BUILD_NUMBER} \
                566540245122.dkr.ecr.ap-south-1.amazonaws.com/prime-clone:${BUILD_NUMBER}

                docker push \
                566540245122.dkr.ecr.ap-south-1.amazonaws.com/prime-clone:latest

                docker push \
                566540245122.dkr.ecr.ap-south-1.amazonaws.com/prime-clone:${BUILD_NUMBER}
                '''
            }
        }

        stage('Update Git Manifest For GitOps') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: "${GITHUB_CRED_ID}",
                        passwordVariable: 'GIT_PASSWORD',
                        usernameVariable: 'GIT_USERNAME'
                    )
                ]) {
                    sh """
                    sed -i "s|image: .*|image: ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com/${IMAGE_REPO_NAME}:${IMAGE_TAG}|g" k8s/deployment.yaml

                    git config user.email "jenkins@devsecops.poc"
                    git config user.name "Jenkins CI Engine"

                    git add k8s/deployment.yaml
                    git commit -m "Automated build update: image tag v${IMAGE_TAG} [skip ci]" || true

                    git remote set-url origin https://\${GIT_USERNAME}:\${GIT_PASSWORD}@github.com/\${GIT_USERNAME}/POC-6.git

                    git push origin HEAD:main
                    """
                }
            }
        }

        stage('Deploy GitOps & Helm Monitoring') {
            steps {
                sh '''
                kubectl create namespace argocd \
                --dry-run=client -o yaml | kubectl apply -f -

                kubectl create namespace monitoring \
                --dry-run=client -o yaml | kubectl apply -f -

                kubectl apply -n argocd \
                -f https://raw.githubusercontent.com/argoproj/argo-cd/v2.11.7/manifests/install.yaml

                kubectl wait \
                --for=condition=available \
                deployment/argocd-server \
                -n argocd \
                --timeout=600s

                kubectl patch svc argocd-server \
                -n argocd \
                -p '{"spec":{"type":"NodePort"}}'

                kubectl apply -f k8s/argocd-app.yaml || true

                helm repo add prometheus-community \
                https://prometheus-community.github.io/helm-charts

                helm repo update

                helm upgrade --install kube-stack \
                prometheus-community/kube-prometheus-stack \
                --namespace monitoring \
                --set grafana.service.type=NodePort \
                --set prometheus.service.type=NodePort \
                --wait \
                --timeout 15m

                kubectl get pods -A
                kubectl get svc -A
                '''
            }
        }

        stage('Display Live Entry Details') {
            steps {
                sh '''
                echo "==============================================="

                NODE_IP=$(aws ec2 describe-instances \
                --filters "Name=instance-state-name,Values=running" \
                --query "Reservations[*].Instances[*].PublicIpAddress" \
                --output text | head -n1)

                ARGOCD_PORT=$(kubectl get svc argocd-server -n argocd -o jsonpath="{.spec.ports[0].nodePort}" || echo "N/A")

                GRAFANA_PORT=$(kubectl get svc kube-stack-grafana -n monitoring -o jsonpath="{.spec.ports[0].nodePort}" || echo "N/A")

                ARGOCD_PASS=$(kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d)

                GRAFANA_PASS=$(kubectl get secret -n monitoring kube-stack-grafana -o jsonpath="{.data.admin-password}" | base64 -d)

                echo "ArgoCD URL    : http://${NODE_IP}:${ARGOCD_PORT}"
                echo "ArgoCD User   : admin"
                echo "ArgoCD Pass   : ${ARGOCD_PASS}"

                echo "Grafana URL   : http://${NODE_IP}:${GRAFANA_PORT}"
                echo "Grafana User  : admin"
                echo "Grafana Pass  : ${GRAFANA_PASS}"

                echo "==============================================="
                '''
            }
        }
    }
}

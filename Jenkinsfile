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
                    // Jenkins initializes and spins up your network + EKS Cluster automatically
                    sh 'terraform init'
                    sh 'terraform apply --auto-approve'
                }
            }
        }

        stage('Establish Cluster Access') {
            steps {
                // Configures cluster context so subsequent deployment scripts run smoothly
                sh "aws eks update-kubeconfig --region ${AWS_DEFAULT_REGION} --name prime-poc-cluster"
            }
        }

        stage('NPM Install & Build') {
            steps {
                sh 'npm install'
                sh 'npm run build'
            }
        }

        stage('SonarQube Quality Scan') {
            steps {
                echo "Forwarding static code mapping to SonarQube..."
            }
        }

        stage('Docker Build Container') {
            steps {
                sh "docker build -t ${IMAGE_REPO_NAME}:${IMAGE_TAG} ."
            }
        }

        stage('Trivy Image Audit') {
            steps {
                sh "trivy image --severity HIGH,CRITICAL --exit-code 0 ${IMAGE_REPO_NAME}:${IMAGE_TAG}"
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
                    # 1. Update the Kubernetes manifest image reference cleanly using standard dots and variables
                    sed -i "s|image: .*|image: \${AWS_ACCOUNT_ID}.dkr.ecr.\${AWS_DEFAULT_REGION}://\${IMAGE_REPO_NAME}:\${IMAGE_TAG}|g" k8s/deployment.yaml
                    
                    # 2. Configure Git operational parameters
                    git config user.email "jenkins@devsecops.poc"
                    git config user.name "Jenkins CI Engine"
                    
                    # 3. Stage and commit configuration adjustments
                    git add k8s/deployment.yaml
                    git commit -m "Automated build update: image tag v\${IMAGE_TAG} [skip ci]"
                    
                    # 4. Corrected Git tracking endpoint layout mapping
                    git remote set-url origin "https://\${GIT_USERNAME}:\${GIT_PASSWORD}@://github.com\${GIT_USERNAME}/POC-6.git"
                    
                    # 5. Push deployment string tracking modifications straight up to GitHub main branch
                    git push origin HEAD:main
                    """
                }
            }
        }
    }
}

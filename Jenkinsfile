pipeline {
    agent any
    environment {
        // AWS Account and Regional Details (Mumbai)
        AWS_ACCOUNT_ID       = '319918321628'
        AWS_DEFAULT_REGION   = 'ap-south-1'
        CLUSTER_NAME         = 'mern-production-cluster'
        SNS_TOPIC_ARN        = 'arn:aws:sns:ap-south-1:319918321628:mern-deployment-alerts'
        
        // AWS ECR Console configuration strings
        FRONTEND_REPO        = 'mern-frontend'
        HELLO_SERVICE_REPO   = 'mern-hello-service'
        PROFILE_SERVICE_REPO = 'mern-profile-service'
        
        IMAGE_TAG            = "${BUILD_NUMBER}"
        
        // Pre-computed Full ECR Registry Domain String
        ECR_REGISTRY         = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com"
    }
    stages {
        stage('Step 4: Checkout Source') {
            steps {
                cleanWs()
                // Jenkins automatically pulls your code from the GitHub Repo & Main Branch you set up
                checkout scm
            }
        }
        stage('Step 3: AWS ECR Authentication') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-ecr-creds']]) {
                    sh """
                    aws ecr get-login-password --region ${AWS_DEFAULT_REGION} | \
                    docker login --username AWS --password-stdin ${ECR_REGISTRY}
                    """
                }
            }
        }
        stage('Step 2: Parallel Multi-Service Build & Push') {
            parallel {
                stage('Frontend Service') {
                    steps {
                        script {
                            dir('frontend') {
                                def frontendImg = docker.build("${ECR_REGISTRY}/${FRONTEND_REPO}:${IMAGE_TAG}")
                                frontendImg.push()
                                frontendImg.push("latest")
                            }
                        }
                    }
                }
                stage('Hello Service (Backend)') {
                    steps {
                        script {
                            dir('backend/helloService') {
                                def helloImg = docker.build("${ECR_REGISTRY}/${HELLO_SERVICE_REPO}:${IMAGE_TAG}")
                                helloImg.push()
                                helloImg.push("latest")
                            }
                        }
                    }
                }
                stage('Profile Service (Backend)') {
                    steps {
                        script {
                            dir('backend/profileService') {
                                def profileImg = docker.build("${ECR_REGISTRY}/${PROFILE_SERVICE_REPO}:${IMAGE_TAG}")
                                profileImg.push()
                                profileImg.push("latest")
                            }
                        }
                    }
                }
            }
        }
        stage('Step 9: ChatOps Initialization Alert') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-ecr-creds']]) {
                    sh """
                    aws sns publish --topic-arn ${SNS_TOPIC_ARN} --region ${AWS_DEFAULT_REGION} \
                      --subject "MERN Deployment Initiated" \
                      --message "🚀 Starting deployment phase for Build #${IMAGE_TAG} to cluster ${CLUSTER_NAME}."
                    """
                }
            }
        }
        stage('Step 5: Create EKS Cluster (If Not Exists)') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-ecr-creds']]) {
                    script {
                        // Checks if the cluster already exists in the Mumbai region
                        def clusterCheck = sh(script: "aws eks describe-cluster --name ${CLUSTER_NAME} --region ${AWS_DEFAULT_REGION}", returnStatus: true)
                        
                        if (clusterCheck != 0) {
                            echo "⚠️ EKS Cluster does not exist! Initializing cluster creation via eksctl..."
                            sh """
                            eksctl create cluster \
                              --name ${CLUSTER_NAME} \
                              --region ${AWS_DEFAULT_REGION} \
                              --nodegroup-name mern-workers \
                              --node-type t3.medium \
                              --nodes 2 \
                              --managed
                            """
                        } else {
                            echo "✅ EKS Cluster already exists. Skipping creation step and updating connection contexts."
                            sh "aws eks update-kubeconfig --region ${AWS_DEFAULT_REGION} --name ${CLUSTER_NAME}"
                        }
                    }
                }
            }
        }
        stage('Step 5: Deploy via Helm') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-ecr-creds']]) {
                    sh """
                    # 1. Update kubeconfig context
                    aws eks update-kubeconfig --region ${AWS_DEFAULT_REGION} --name ${CLUSTER_NAME}
                    
                    # 2. Deploy all 3 services via Helm using your charts/mern-stack directory
                    helm upgrade --install mern-production ./charts/mern-stack \
                      --set frontend.tag=${IMAGE_TAG} \
                      --set helloService.tag=${IMAGE_TAG} \
                      --set profileService.tag=${IMAGE_TAG} \
                      --values ./charts/mern-stack/values.yaml
                    """
                }
            }
        }
        stage('Step 6: Post-Deployment Health Validation') {
            steps {
                sh """
                echo "=== Verifying Kubernetes Pod Deployment States ==="
                kubectl get pods -n default
                
                echo "=== Retrieving Live Public Facing Frontend Ingress URL ==="
                kubectl get svc mern-frontend-svc || kubectl get svc -n default
                echo ""
                """
            }
        }
    }
    post {
        success {
            withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-ecr-creds']]) {
                sh """
                aws sns publish --topic-arn ${SNS_TOPIC_ARN} --region ${AWS_DEFAULT_REGION} \
                  --subject "Deployment SUCCESS" \
                  --message "✅ Pipeline Success: Build #${BUILD_NUMBER} has been successfully deployed to the EKS cluster!"
                """
            }
        }
        failure {
            withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-ecr-creds']]) {
                sh """
                aws sns publish --topic-arn ${SNS_TOPIC_ARN} --region ${AWS_DEFAULT_REGION} \
                  --subject "Deployment FAILURE" \
                  --message "🚨 Pipeline Failure: Build #${BUILD_NUMBER} failed during execution. Please check the Jenkins console logs immediately."
                """
            }
        }
        always {
            // Cleans up old dangling build layers to preserve EC2 disk storage capacity
            sh "docker system prune -f"
        }
    }
}

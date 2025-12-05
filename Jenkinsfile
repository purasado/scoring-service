pipeline {
    agent any

    environment {
        AWS_REGION = "ap-south-1"
        REPO_NAME  = "scoring-service"
        IMAGE_TAG  = "latest"
    }

    stages {
        stage('Checkout') {
            steps { checkout scm }
        }

        stage('Unit Tests') {
            steps {
                sh '''
                python3 -m venv venv
                . venv/bin/activate
                pip install -r requirements.txt pytest
                pytest -q
                '''
            }
        }

        stage('Build Docker') {
    steps {
        sh """
        export AWS_PAGER=""
        docker build -t scoring-service:latest .
        """
    }
}
        stage('Init AWS') {
    steps {
        script {
            env.ACCOUNT_ID = sh(
                script: "AWS_PAGER='' aws sts get-caller-identity --query Account --output text",
                returnStdout: true
            ).trim()
            echo "Using AWS Account: ${env.ACCOUNT_ID}"
        }
    }
}



        stage('Push to ECR') {
    steps {
        sh """
        set -e
        export AWS_PAGER=""

        # Get AWS account ID
        ACCOUNT_ID=\$(aws sts get-caller-identity --query Account --output text)
        echo "Using AWS Account: \$ACCOUNT_ID"

        # ECR login
        aws ecr get-login-password --region ${AWS_REGION} \
          | docker login --username AWS --password-stdin \$ACCOUNT_ID.dkr.ecr.${AWS_REGION}.amazonaws.com

        # Tag & push image to ECR
        docker tag scoring-service:latest \$ACCOUNT_ID.dkr.ecr.${AWS_REGION}.amazonaws.com/${REPO_NAME}:${IMAGE_TAG}
        docker push \$ACCOUNT_ID.dkr.ecr.${AWS_REGION}.amazonaws.com/${REPO_NAME}:${IMAGE_TAG}
        """
    }
}

        stage('Deploy to K8s') {
    steps {
        sh """
        set -e
        export AWS_PAGER=""

        # Configure kubeconfig for EKS
        aws eks update-kubeconfig --region ${AWS_REGION} --name scoring-eks

        echo "Current kube context:"
        kubectl config current-context

        echo "Cluster nodes:"
        kubectl get nodes

        # Update deployment image
        kubectl set image deployment/scoring-deployment \
          scoring-container=680993827964.dkr.ecr.${AWS_REGION}.amazonaws.com/scoring-service:${IMAGE_TAG} \
          --namespace default

        # Wait for rollout
        kubectl rollout status deployment/scoring-deployment --namespace default
        """
    }
}

    }
}

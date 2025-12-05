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
                kubectl set image deployment/scoring-deployment scoring-container=${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${REPO_NAME}:${IMAGE_TAG} --namespace default
                kubectl rollout status deployment/scoring-deployment --namespace default
                """
            }
        }
    }
}

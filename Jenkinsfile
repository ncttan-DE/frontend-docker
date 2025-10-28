pipeline {
    agent any

    environment {
        IMAGE_TAG = "${env.BUILD_NUMBER}"
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/ncttan-DE/frontend-docker'
            }
        }

        stage('Build & Push to ECR') {
            steps {
                withCredentials([
                    string(credentialsId: 'aws-account-id', variable: 'AWS_ACCOUNT_ID'),
                    string(credentialsId: 'aws-region', variable: 'AWS_REGION'),
                    [$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-ecr-cred']
                ]) {
                    sh '''
                        echo "🔧 Building Docker image..."
                        docker build -t $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/frontend-docker:$BUILD_NUMBER .

                        echo "🔐 Logging in to ECR..."
                        aws ecr get-login-password --region $AWS_REGION | \
                        docker login --username AWS --password-stdin $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com

                        echo "🚀 Pushing image to ECR..."
                        docker push $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/frontend-docker:$BUILD_NUMBER

                        echo "🏷️ Tagging image as latest..."
                        docker tag $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/frontend-docker:$BUILD_NUMBER \
                                   $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/frontend-docker:latest

                        docker push $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/frontend-docker:latest
                    '''
                }
            }
        } 
    }

    post {
        success {
            echo "✅ Successfully built and pushed image to ECR: $IMAGE_TAG"
        }
        failure {
            echo "❌ Build failed. Check Jenkins logs for details."
        }
    }
}

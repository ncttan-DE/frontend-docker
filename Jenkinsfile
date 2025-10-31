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
        stage('Clean Up') {
            steps {
                sh 'docker rmi $(docker images -f "dangling=true" -q) || true'
            }
        }

        stage('Notify Success') {
            steps {
                echo "✅ Docker image built and pushed successfully: $IMAGE_TAG"
            }
        }

        stage('cleanup git repo') {
            steps {
                sh 'rm -rf k8s-cd'
            }
        }
        
        stage('Update Manifest') {
            steps {
                withCredentials([
                    string(credentialsId: 'github-k8s', variable: 'GITHUB_K8S'),
                    string(credentialsId: 'aws-account-id', variable: 'AWS_ACCOUNT_ID'),
                    string(credentialsId: 'aws-region', variable: 'AWS_REGION')
                ]) {
                    sh "git clone https://${GITHUB_K8S}"
                            
                    dir('k8s-cd/backend') {
                        sh """
                        sed -i 's#image: .*\$#image: ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/frontend-docker:${IMAGE_TAG}#' frontend.yaml

                        git config user.email "jenkins@ci.com"
                        git config user.name "Jenkins CI"
                        git commit -am "Update image tag to ${IMAGE_TAG}"
                        """
                    }

                    withCredentials([usernamePassword(credentialsId: 'github-token', usernameVariable: 'GIT_USER', passwordVariable: 'GIT_TOKEN')]) {
                        sh """
                        cd k8s-cd
                        git push https://${GIT_USER}:${GIT_TOKEN}@${GITHUB_K8S} main
                        """
                    }
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

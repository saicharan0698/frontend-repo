pipeline {
  agent any

  environment {
    AWS_REGION   = 'us-east-1'
    ECR_REGISTRY = '223425500491.dkr.ecr.us-east-1.amazonaws.com'
    ECR_REPO     = 'frontend-repo'
    EKS_CLUSTER  = 'my-eks-cluster'
    IMAGE_TAG    = "${env.GIT_COMMIT[0..6]}"
    IMAGE_FULL   = "${ECR_REGISTRY}/${ECR_REPO}:${IMAGE_TAG}"
  }

  stages {

    stage('Checkout') {
      steps {
        git branch: 'master', url: 'https://github.com/saicharan0698/frontend-repo'
      }
    }

    stage('Build Docker Image') {
      steps {
        sh "docker build -t ${IMAGE_FULL} ."
      }
    }

    stage('Push to ECR') {
      steps {
        withCredentials([
          string(credentialsId: 'AWS_ACCESS_KEY_ID',     variable: 'AWS_ACCESS_KEY_ID'),
          string(credentialsId: 'AWS_SECRET_ACCESS_KEY', variable: 'AWS_SECRET_ACCESS_KEY')
        ]) {
          sh '''
            export AWS_ACCESS_KEY_ID=$AWS_ACCESS_KEY_ID
            export AWS_SECRET_ACCESS_KEY=$AWS_SECRET_ACCESS_KEY
            export AWS_DEFAULT_REGION=$AWS_REGION

            aws ecr get-login-password --region $AWS_REGION | \
            docker login --username AWS --password-stdin $ECR_REGISTRY

            docker push $IMAGE_FULL
          '''
        }
      }
    }

    stage('Deploy to Dev') {
      steps {
        withCredentials([
          string(credentialsId: 'AWS_ACCESS_KEY_ID',     variable: 'AWS_ACCESS_KEY_ID'),
          string(credentialsId: 'AWS_SECRET_ACCESS_KEY', variable: 'AWS_SECRET_ACCESS_KEY')
        ]) {
          sh '''
            export AWS_ACCESS_KEY_ID=$AWS_ACCESS_KEY_ID
            export AWS_SECRET_ACCESS_KEY=$AWS_SECRET_ACCESS_KEY
            export AWS_DEFAULT_REGION=$AWS_REGION

            cp ./helm/values-dev.yaml ./helm/values.yaml
            sed -i "s|REPLACE_IMAGE_REPO|$ECR_REGISTRY/$ECR_REPO|g" ./helm/values.yaml
            sed -i "s|REPLACE_IMAGE_TAG|$IMAGE_TAG|g"               ./helm/values.yaml

            aws eks update-kubeconfig --region $AWS_REGION --name $EKS_CLUSTER
            aws --version

            aws sts get-caller-identity
            
            kubectl config current-context
            
            kubectl get nodes
            
            kubectl auth can-i '*' '*' --all-namespaces
            helm upgrade --install frontend-dev ./helm \
              --values ./helm/values.yaml \
              --wait
          '''
        }
      }
    }
  }

  post {
    success { echo "Pipeline completed. Image: ${IMAGE_FULL}" }
    failure { echo 'Pipeline failed — check logs above' }
  }
}

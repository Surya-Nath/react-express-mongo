pipeline {
  agent any
  environment {
    AWS_REGION = 'ap-south-1'
    REGISTRY   = '413816840602.dkr.ecr.ap-south-1.amazonaws.com'
  }
  stages {
    stage('Checkout') { steps { checkout scm } }
    stage('ECR login') {
      steps {
        withCredentials([usernamePassword(
          credentialsId: 'aws-ecr',
          usernameVariable: 'AWS_ACCESS_KEY_ID',
          passwordVariable: 'AWS_SECRET_ACCESS_KEY'
        )]) {
          sh '''
            aws ecr get-login-password --region "$AWS_REGION" \
              | docker login --username AWS --password-stdin "$REGISTRY"
          '''
        }
      }
    }
    stage('UI') {
      steps {
        sh '''
          docker build -t $REGISTRY/week4-ui:$GIT_COMMIT -t $REGISTRY/week4-ui:latest ./frontend
          docker push $REGISTRY/week4-ui:$GIT_COMMIT
          docker push $REGISTRY/week4-ui:latest
        '''
      }
    }
    stage('API') {
      steps {
        sh '''
          docker build -t $REGISTRY/week4-api:$GIT_COMMIT -t $REGISTRY/week4-api:latest ./backend
          docker push $REGISTRY/week4-api:$GIT_COMMIT
          docker push $REGISTRY/week4-api:latest
        '''
      }
    }
  }
}

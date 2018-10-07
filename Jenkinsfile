
pipeline {
  environment{
    DOCKER_PASSWORD  = credentials('docker_password')
    AWS_ACCESS_KEY_ID     = credentials('aws-secret-key-id')
    AWS_SECRET_ACCESS_KEY = credentials('aws-secret-access-key')
  }
  agent any
  stages {
    stage('Checkout Code') {
      steps {
        checkout scm
      }
    }
    stage('Build Catalogue Image') {
      steps {
          sh '''
              docker login -u davitron -p ${DOCKER_PASSWORD}
              cd ${WORKSPACE}/docker/catalogue
              IMAGE="catalogue"
              docker build --no-cache -t ${IMAGE}:${BUILD_NUMBER} .
            '''
          }
    }
    stage('Build Catalogue-db Image') {
      steps {
          sh '''
              docker login -u davitron -p ${DOCKER_PASSWORD}
              cd ${WORKSPACE}/docker/catalogue-db
              IMAGE="catalogue-db"
              docker build --no-cache -t ${IMAGE}:${BUILD_NUMBER} .
            '''
          }
    }
    stage('Build Nginx Image') {
      steps {
          sh '''
              docker login -u davitron -p ${DOCKER_PASSWORD}
              cd ${WORKSPACE}/docker/nginx
              IMAGE="nginx-router"
              docker build --no-cache -t ${IMAGE}:${BUILD_NUMBER} .
            '''
          }
    }
    stage('Push image to AWS ECR') {
      agent { label 'master' }
      steps {
          sh '''
              aws ecr get-login --no-include-email --region us-east-2 | bash
              ECR_ADDRESS="770817924447.dkr.ecr.us-east-2.amazonaws.com"
              IMAGE1="catalogue"
              IMAGE2="catalog-db"
              IMAGE3="nginx-router"
              docker tag ${IMAGE1}:${BUILD_NUMBER} ${ECR_ADDRESS}/${IMAGE1}:${BUILD_NUMBER}
              docker tag ${IMAGE2}:${BUILD_NUMBER} ${ECR_ADDRESS}/${IMAGE2}:${BUILD_NUMBER}
              docker tag ${IMAGE3}:${BUILD_NUMBER} ${ECR_ADDRESS}/${IMAGE3}:${BUILD_NUMBER}
              docker push ${ECR_ADDRESS}/${IMAGE1}:${BUILD_NUMBER}
              docker push ${ECR_ADDRESS}/${IMAGE2}:${BUILD_NUMBER}
              docker push ${ECR_ADDRESS}/${IMAGE3}:${BUILD_NUMBER}
            '''
      }
    }
    stage('Deploy with Kubernetes') {
      agent { label 'master' }
      steps {
        sh '''
          NEW_ECR_IMAGE1="726336258647.dkr.ecr.us-east-2.amazonaws.com/catalogue:${BUILD_NUMBER}"
          NEW_ECR_IMAGE2="726336258647.dkr.ecr.us-east-2.amazonaws.com/catalogue-db:${BUILD_NUMBER}"
          NEW_ECR_IMAGE3="726336258647.dkr.ecr.us-east-2.amazonaws.com/nginx-router:${BUILD_NUMBER}"
          kubectl set image deployment/catalogue-deployment catalogue=$NEW_DOCKER_IMAGE1
          kubectl set image deployment/catalogue-db-deployment catalogue-db=$NEW_DOCKER_IMAGE2
          kubectl set image deployment/nginx-router nginx-router=$NEW_DOCKER_IMAGE3
          kubectl rollout status deployment catalogue-deployment
          kubectl rollout status deployment catalogue-db-deployment
          kubectl rollout status deployment nginx-router
          '''
      }
    }
  }
}


pipeline {
  agent any

  environment {
    DOCKERHUB_USER = '' // set at job level or here
    IMAGE_NAME     = "${DOCKERHUB_USER}/cicd-demo"
    SONARQUBE_ENV  = 'MySonarQube'
  }

  stages {
    stage('Checkout') {
      steps { checkout scm }
    }

    stage('SonarQube Scan') {
  steps {
    withSonarQubeEnv("${SONARQUBE_ENV}") {
      withCredentials([string(credentialsId: 'SONAR_TOKEN', variable: 'SONAR_TOKEN')]) {
        sh '''
          export SONAR_TOKEN="$SONAR_TOKEN"
          /var/jenkins_home/tools/hudson.plugins.sonar.SonarRunnerInstallation/SonarQubeScanner/bin/sonar-scanner \
            -Dsonar.projectKey=cicd-pipeline \
            -Dsonar.sources=app
        '''
      }
    }
  }
}



    stage('Build Image') {
      steps {
        sh '''
          docker build -t $IMAGE_NAME:${BUILD_NUMBER} -f docker/Dockerfile .
          docker tag $IMAGE_NAME:${BUILD_NUMBER} $IMAGE_NAME:latest
        '''
      }
    }

    stage('Push Image') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'DOCKERHUB_CREDS', usernameVariable: 'USER', passwordVariable: 'PASS')]) {
          sh '''
            echo "$PASS" | docker login -u "$USER" --password-stdin
            docker push $IMAGE_NAME:${BUILD_NUMBER}
            docker push $IMAGE_NAME:latest
          '''
        }
      }
    }

    stage('Deploy to Kubernetes') {
      steps {
        sh '''
          # update image tag in deployment
          sed -i "s|image: .*$|image: $IMAGE_NAME:${BUILD_NUMBER}|" k8s/deployment.yaml || true
          kubectl apply -f k8s/
          kubectl rollout status deployment/cicd-demo --timeout=120s
        '''
      }
    }
  }

  post {
    success { echo ' Deploy succeeded' }
    failure { echo ' Build failed' }
  }
}           

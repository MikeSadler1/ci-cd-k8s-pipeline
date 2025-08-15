pipeline {
  agent any

  environment {
    DOCKERHUB_USER = "${env.DOCKERHUB_USER ?: 'mikesadler1'}"
    IMAGE_NAME     = "${env.DOCKERHUB_USER ?: 'mikesadler1'}/cicd-pipeline"
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
            // Use the configured scanner tool if you have it set up in Jenkins (recommended):
            withEnv(["PATH+SONAR=${tool 'SonarQubeScanner'}/bin"]) {
              sh '''
                export SONAR_TOKEN="$SONAR_TOKEN"
                sonar-scanner \
                  -Dsonar.projectKey=cicd-pipeline \
                  -Dsonar.sources=app
              '''
            }
            // If you prefer the absolute path, comment the block above and uncomment below:
            // sh '''
            //   export SONAR_TOKEN="$SONAR_TOKEN"
            //   /var/jenkins_home/tools/hudson.plugins.sonar.SonarRunnerInstallation/SonarQubeScanner/bin/sonar-scanner \
            //     -Dsonar.projectKey=cicd-pipeline \
            //     -Dsonar.sources=app
            // '''
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
          # update image tag in deployment (note the capital K in K8s/)
          sed -i "s#image: .*#image: $IMAGE_NAME:${BUILD_NUMBER}#" K8s/deployment.yaml || true

          kubectl apply -f K8s/
          kubectl rollout status deployment/cicd-pipeline --timeout=120s
        '''
      }
    }
  }

  post {
    success { echo 'Deploy succeeded' }
    failure { echo  'Build failed' }
  }
}
           

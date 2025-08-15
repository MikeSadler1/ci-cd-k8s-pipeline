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

    // --- New: make sure kubectl (correct arch) is available in this build ---
    stage('Kubectl Setup') {
      steps {
        sh '''
          set -e
          mkdir -p "$WORKSPACE/bin"
          if ! command -v kubectl >/dev/null 2>&1; then
            ARCH=$(uname -m)
            case "$ARCH" in
              aarch64) KARCH=arm64 ;;
              x86_64)  KARCH=amd64 ;;
              *)       KARCH=amd64 ;;
            esac
            # Grab a recent kubectl; change version if you want
            curl -fsSL -o "$WORKSPACE/bin/kubectl" \
              https://dl.k8s.io/release/$(curl -fsSL https://dl.k8s.io/release/stable.txt)/bin/linux/${KARCH}/kubectl
            chmod +x "$WORKSPACE/bin/kubectl"
          fi
          "$WORKSPACE/bin/kubectl" version --client --short || true
        '''
      }
    }

    stage('SonarQube Scan') {
      steps {
        withSonarQubeEnv("${SONARQUBE_ENV}") {
          withCredentials([string(credentialsId: 'SONAR_TOKEN', variable: 'SONAR_TOKEN')]) {
            // Use the configured scanner tool if you have it set up in Jenkins (recommended):
            withEnv(["PATH+SONAR=${tool 'SonarQubeScanner'}/bin"]) {
              sh '''
                set -e
                sonar-scanner \
                  -Dsonar.projectKey=cicd-pipeline \
                  -Dsonar.sources=app \
                  -Dsonar.token="$SONAR_TOKEN"
              '''
            }
            // If you prefer an absolute path for the scanner, use this instead:
            // sh '''
            //   set -e
            //   /var/jenkins_home/tools/hudson.plugins.sonar.SonarRunnerInstallation/SonarQubeScanner/bin/sonar-scanner \
            //     -Dsonar.projectKey=cicd-pipeline \
            //     -Dsonar.sources=app \
            //     -Dsonar.token="$SONAR_TOKEN"
            // '''
          }
        }
      }
    }

    stage('Build Image') {
      steps {
        sh '''
          set -e
          docker build -t "$IMAGE_NAME:${BUILD_NUMBER}" -f docker/Dockerfile .
          docker tag "$IMAGE_NAME:${BUILD_NUMBER}" "$IMAGE_NAME:latest"
        '''
      }
    }

    stage('Push Image') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'DOCKERHUB_CREDS', usernameVariable: 'USER', passwordVariable: 'PASS')]) {
          sh '''
            set -e
            echo "$PASS" | docker login -u "$USER" --password-stdin
            docker push "$IMAGE_NAME:${BUILD_NUMBER}"
            docker push "$IMAGE_NAME:latest"
          '''
        }
      }
    }

    stage('Deploy to Kubernetes') {
  steps {
    sh '''
      set -e
      export PATH="$WORKSPACE/bin:$PATH"

      # Update image tag in manifest (note the capital K in K8s/)
      sed -i "s#image: .*#image: $IMAGE_NAME:${BUILD_NUMBER}#" K8s/deployment.yaml || true

      # Find host-forwarded port for Minikube API (8443 in the minikube container)
      API_PORT="$(docker inspect -f '{{(index (index .NetworkSettings.Ports "8443/tcp") 0).HostPort}}' minikube)"
      KUBE_SERVER="https://host.docker.internal:${API_PORT}"

      # Apply and wait (override server & skip TLS hostname mismatch)
      kubectl --kubeconfig=/var/jenkins_home/.kube/config \
              --server="$KUBE_SERVER" \
              --insecure-skip-tls-verify=true \
              apply -f K8s/

      kubectl --kubeconfig=/var/jenkins_home/.kube/config \
              --server="$KUBE_SERVER" \
              --insecure-skip-tls-verify=true \
              rollout status deployment/cicd-pipeline --timeout=120s
    '''
  }
}


  post {
    success { echo 'Deploy succeeded' }
    failure { echo 'Build failed' }
  }
}

              

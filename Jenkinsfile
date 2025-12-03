pipeline {
  agent { label 'test-jenkins-cluster' }

  environment {
    IMAGE_NAME = "busybox"
    IMAGE_TAG  = "latest"
    NAMESPACE  = "test-ns"
    MANIFEST   = "k8s-busybox-http-${IMAGE_TAG}.yaml"
    KUBECONFIG_CRED = "kubeconfig-file-test-cluster"
  }

  stages {

    stage('Checkout') {
      steps { checkout scm }
    }

    stage('1 - Pull image from registry (optional)') {
      steps {
        sh '''
          set -eux || true
          echo "No docker available — cluster will pull image"
        '''
      }
    }

    stage('2 - Generate manifest at runtime') {
      steps {
        sh """
          set -eux
          cat > ${MANIFEST} <<'EOF'
          ... (your manifest content unchanged)
          EOF
        """
      }
    }

    stage('3 - kubectl apply') {
      steps {
        withCredentials([file(credentialsId: env.KUBECONFIG_CRED, variable: 'KUBECONFIG_FILE')]) {
          container('kubectl') {
            sh '''
              set -eux
              export KUBECONFIG="${KUBECONFIG_FILE}"
              kubectl get namespace ${NAMESPACE} >/dev/null 2>&1 || kubectl create namespace ${NAMESPACE}
              kubectl apply -n ${NAMESPACE} -f ${MANIFEST}
            '''
          }
        }
      }
    }

    stage('4 - Rollout verification') {
      steps {
        withCredentials([file(credentialsId: env.KUBECONFIG_CRED, variable: 'KUBECONFIG_FILE')]) {
          container('kubectl') {
            sh '''
              set -eux
              export KUBECONFIG="${KUBECONFIG_FILE}"
              kubectl rollout status deployment/busybox-http -n ${NAMESPACE} --timeout=120s
            '''
          }
        }
      }
    }

    stage('5 - Pod validation using kubectl get pods') {
      steps {
        withCredentials([file(credentialsId: env.KUBECONFIG_CRED, variable: 'KUBECONFIG_FILE')]) {
          container('kubectl') {
            sh '''
              set -eux
              export KUBECONFIG="${KUBECONFIG_FILE}"
              kubectl get pods -n ${NAMESPACE} -o wide
            '''
          }
        }
      }
    }

    stage('6 - Basic post-deploy test using curl') {
      steps {
        withCredentials([file(credentialsId: env.KUBECONFIG_CRED, variable: 'KUBECONFIG_FILE')]) {
          container('kubectl') {
            sh '''
              set -eux
              export KUBECONFIG="${KUBECONFIG_FILE}"
              PODNAME="curl-test-$(date +%s)"
              kubectl run "${PODNAME}" --rm -i --restart=Never --image=curlimages/curl -n ${NAMESPACE} --command -- sh -c '
                for i in 1 2 3 4 5; do
                  if curl -sS -m 5 http://busybox-http-svc/; then exit 0; fi
                  sleep 2
                done
                exit 1
              '
            '''
          }
        }
      }
    }

    stage('7 - Cleanup') {
      steps {
        withCredentials([file(credentialsId: env.KUBECONFIG_CRED, variable: 'KUBECONFIG_FILE')]) {
          container('kubectl') {
            sh '''
              set -eux || true
              export KUBECONFIG="${KUBECONFIG_FILE}"
              kubectl delete -n ${NAMESPACE} -f ${MANIFEST} --ignore-not-found=true
            '''
          }
        }
      }
    }

  } // stages

  post {
    always {
      archiveArtifacts artifacts: "${MANIFEST}", fingerprint: true
    }
  }

}

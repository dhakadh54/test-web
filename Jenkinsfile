pipeline {
  agent { label 'test-cluster' }
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

    stage('1 - Pull image from registry (optional local pull)') {
      steps {
        script {
          sh '''
            set -eux || true
            if command -v docker >/dev/null 2>&1; then
              echo "Agent has docker CLI — attempting docker pull ${IMAGE_NAME}:${IMAGE_TAG}"
              docker pull ${IMAGE_NAME}:${IMAGE_TAG} || true
            else
              echo "No docker CLI on agent — cluster will pull image"
            fi
          '''
        }
      }
    }

    stage('2 - Generate manifest at runtime') {
      steps {
        script {
          sh """
            set -eux
            cat > ${MANIFEST} <<'EOF'
            apiVersion: apps/v1
            kind: Deployment
            metadata:
              name: busybox-http
              labels:
                app: busybox-http
            spec:
              replicas: 1
              selector:
                matchLabels:
                  app: busybox-http
              template:
                metadata:
                  labels:
                    app: busybox-http
                spec:
                  containers:
                    - name: busybox
                      image: ${IMAGE_NAME}:${IMAGE_TAG}
                      command: ["/bin/sh","-c"]
                      args:
                        - mkdir -p /www && echo "Hello from BusyBox (deployed via Jenkins)" > /www/index.html && httpd -f -p 8080 -h /www
                      ports:
                        - containerPort: 8080
                      readinessProbe:
                        httpGet:
                          path: /
                          port: 8080
                        initialDelaySeconds: 2
                        periodSeconds: 3
            ---
            apiVersion: v1
            kind: Service
            metadata:
              name: busybox-http-svc
            spec:
              selector:
                app: busybox-http
              ports:
                - protocol: TCP
                  port: 80
                  targetPort: 8080
            EOF
            echo "Manifest generated: ${MANIFEST}"
            sed -n '1,200p' ${MANIFEST} || true
          """
        }
      }
    }

    stage('3 - kubectl apply') {
      steps {
        withCredentials([file(credentialsId: env.KUBECONFIG_CRED, variable: 'KUBECONFIG_FILE')]) {
          sh '''
            set -eux
            export KUBECONFIG="${KUBECONFIG_FILE}"
            kubectl get namespace ${NAMESPACE} >/dev/null 2>&1 || kubectl create namespace ${NAMESPACE}
            kubectl apply -n ${NAMESPACE} -f ${MANIFEST}
          '''
        }
      }
    }

    stage('4 - Rollout verification') {
      steps {
        withCredentials([file(credentialsId: env.KUBECONFIG_CRED, variable: 'KUBECONFIG_FILE')]) {
          sh '''
            set -eux
            export KUBECONFIG="${KUBECONFIG_FILE}"
            kubectl rollout status deployment/busybox-http -n ${NAMESPACE} --timeout=120s
          '''
        }
      }
    }

    stage('5 - Pod validation using kubectl get pods') {
      steps {
        withCredentials([file(credentialsId: env.KUBECONFIG_CRED, variable: 'KUBECONFIG_FILE')]) {
          sh '''
            set -eux
            export KUBECONFIG="${KUBECONFIG_FILE}"
            kubectl get pods -n ${NAMESPACE} -o wide
          '''
        }
      }
    }

    stage('6 - Basic post-deploy test using curl (in-cluster)') {
      steps {
        withCredentials([file(credentialsId: env.KUBECONFIG_CRED, variable: 'KUBECONFIG_FILE')]) {
          sh '''
            set -eux
            export KUBECONFIG="${KUBECONFIG_FILE}"
            PODNAME="curl-test-$(date +%s)"
            kubectl run "${PODNAME}" --rm -i --restart=Never --image=curlimages/curl -n ${NAMESPACE} --command -- sh -c '
              set -eux
              for i in 1 2 3 4 5; do
                echo "Attempt ${i}: curl -sS -m 5 http://busybox-http-svc/"
                if curl -sS -m 5 http://busybox-http-svc/; then
                  echo "Service responded"
                  exit 0
                else
                  echo "Not ready yet; sleeping 2s"
                  sleep 2
                fi
              done
              echo "Service did not respond" >&2
              exit 1
            '
          '''
        }
      }
    }

    stage('7 - Cleanup using kubectl delete -f manifest') {
      steps {
        withCredentials([file(credentialsId: env.KUBECONFIG_CRED, variable: 'KUBECONFIG_FILE')]) {
          sh '''
            set -eux || true
            export KUBECONFIG="${KUBECONFIG_FILE}"
            kubectl delete -n ${NAMESPACE} -f ${MANIFEST} --ignore-not-found=true || true
            echo "Cleanup done."
          '''
        }
      }
    }
  } // stages

  post {
    always {
      archiveArtifacts artifacts: "${MANIFEST}", fingerprint: true, allowEmptyArchive: true
      echo 'Pipeline finished.'
    }
    failure { echo 'Pipeline failed — inspect logs.' }
  }
}

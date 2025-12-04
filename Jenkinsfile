pipeline {
  agent { label 'test-jenkins-cluster1' }

  environment {
    IMAGE_NAME = "busybox"                          // change if needed
    IMAGE_TAG  = "latest"
    NAMESPACE  = "test"
    MANIFEST   = "k8s-${IMAGE_NAME}-${IMAGE_TAG}.yaml"
    // Credential ID you created earlier (Secret file type)
    KUBECONFIG_CRED = "kubeconfig-file-test-cluster"
  }

  options {
    timeout(time: 30, unit: 'MINUTES')
  }

  stages {
    stage('Checkout') {
      steps { checkout scm }
    }

    stage('1 - Pull image from registry (optional)') {
      steps {
        script {
          sh '''
            set -eux || true
            # inside pod you may not have docker; we attempt but ignore failure
            if command -v docker >/dev/null 2>&1; then
              echo "Agent has docker CLI — trying local pull ${IMAGE_NAME}:${IMAGE_TAG}"
              docker pull ${IMAGE_NAME}:${IMAGE_TAG} || true
            else
              echo "No docker CLI in pod — cluster will pull image"
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
              name: ${IMAGE_NAME}-http
              labels:
                app: ${IMAGE_NAME}-http
            spec:
              replicas: 1
              selector:
                matchLabels:
                  app: ${IMAGE_NAME}-http
              template:
                metadata:
                  labels:
                    app: ${IMAGE_NAME}-http
                spec:
                  containers:
                    - name: ${IMAGE_NAME}
                      image: ${IMAGE_NAME}:${IMAGE_TAG}
                      command: ["/bin/sh","-c"]
                      args:
                        - mkdir -p /www && echo "Hello from ${IMAGE_NAME} (deployed via Jenkins)" > /www/index.html && httpd -f -p 8080 -h /www
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
              name: ${IMAGE_NAME}-http-svc
            spec:
              selector:
                app: ${IMAGE_NAME}-http
              ports:
                - protocol: TCP
                  port: 80
                  targetPort: 8080
            EOF

            echo "Generated manifest: ${MANIFEST}"
            ls -l ${MANIFEST}
            sed -n '1,200p' ${MANIFEST} || true
          """
        }
      }
    }

    stage('3 - kubectl apply') {
      steps {
        // mount kubeconfig secret file into the container
        withCredentials([file(credentialsId: env.KUBECONFIG_CRED, variable: 'KUBECONFIG_FILE')]) {
          sh '''
            set -eux
            export KUBECONFIG="${KUBECONFIG_FILE}"

            # ensure namespace exists (idempotent)
            kubectl get namespace ${NAMESPACE} >/dev/null 2>&1 || kubectl create namespace ${NAMESPACE}

            # apply manifest (manifest is in workspace)
            kubectl apply -n ${NAMESPACE} -f ${WORKSPACE}/${MANIFEST}
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
            kubectl rollout status deployment/${IMAGE_NAME}-http -n ${NAMESPACE} --timeout=120s
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
            echo "Pods in namespace ${NAMESPACE}:"
            kubectl get pods -n ${NAMESPACE} -o wide
          '''
        }
      }
    }

    stage('6 - Run a basic post-deploy test using curl (in-cluster)') {
      steps {
        withCredentials([file(credentialsId: env.KUBECONFIG_CRED, variable: 'KUBECONFIG_FILE')]) {
          sh '''
            set -eux
            export KUBECONFIG="${KUBECONFIG_FILE}"
            PODNAME="curl-test-$(date +%s)"
            # use kubectl to run an ephemeral curl pod inside the cluster (this pod will run in target cluster)
            kubectl run "${PODNAME}" --rm -i --restart=Never --image=curlimages/curl -n ${NAMESPACE} --command -- sh -c '
              set -eux
              for i in 1 2 3 4 5; do
                echo "Attempt ${i}: curl -sS -m 5 http://${IMAGE_NAME}-http-svc/"
                if curl -sS -m 5 http://${IMAGE_NAME}-http-svc/; then
                  echo "Service responded"
                  exit 0
                else
                  echo "Not ready; sleeping 2s"
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

    stage('7 - Cleanup using kubectl delete -f <manifest>') {
      steps {
        withCredentials([file(credentialsId: env.KUBECONFIG_CRED, variable: 'KUBECONFIG_FILE')]) {
          sh '''
            set -eux || true
            export KUBECONFIG="${KUBECONFIG_FILE}"
            kubectl delete -n ${NAMESPACE} -f ${WORKSPACE}/${MANIFEST} --ignore-not-found=true || true
            echo "Cleanup complete."
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
    failure {
      echo 'Pipeline failed — check console output.'
    }
  }
}


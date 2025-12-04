// Jenkinsfile - Declarative pipeline with kubectl docker fallback
pipeline {
  agent { label 'test-jenkins-cluster' }
  environment {
    IMAGE_NAME = "busybox"                      
    IMAGE_TAG  = "latest"
    NAMESPACE  = "jenkins"
    MANIFEST   = "k8s-${IMAGE_NAME}-${IMAGE_TAG}.yaml"
    KUBECONFIG_CRED = "kubeconfig-file-test-cluster" // Secret File credential in Jenkins
    KUBECTL_DOCKER_IMAGE = "bitnami/kubectl:latest"  // image used for fallback
  }

  options {
    timeout(time: 30, unit: 'MINUTES')
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('1 - Pull image from registry (optional)') {
      steps {
        script {
          sh '''
            set -eux || true
            if command -v docker >/dev/null 2>&1; then
              echo "Agent has docker CLI — attempting local pull ${IMAGE_NAME}:${IMAGE_TAG}"
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

            echo "Manifest created: ${MANIFEST}"
            sed -n '1,200p' ${MANIFEST} || true
          """
        }
      }
    }

    // All kubectl usage below uses run_kubectl helper which falls back to docker-run bitnami/kubectl if needed
    stage('3 - kubectl apply') {
      steps {
        withCredentials([file(credentialsId: env.KUBECONFIG_CRED, variable: 'KUBECONFIG_FILE')]) {
          sh '''
            set -eux

            # helper: run_kubectl ARGS...
            # uses local kubectl if available, otherwise runs kubectl inside a docker container
            run_kubectl() {
              if command -v kubectl >/dev/null 2>&1; then
                echo "Using local kubectl"
                kubectl "$@"
              else
                echo "Local kubectl not found — using dockerized kubectl (${KUBECTL_DOCKER_IMAGE})"
                # mount kubeconfig and workspace; pass through all args
                docker run --rm -v "${KUBECONFIG_FILE}:/kubeconfig:ro" -v "${WORKSPACE}:${WORKSPACE}" ${KUBECTL_DOCKER_IMAGE} \
                  kubectl --kubeconfig /kubeconfig "$@"
              fi
            }

            export KUBECONFIG="${KUBECONFIG_FILE}"
            # ensure namespace exists
            run_kubectl get namespace ${NAMESPACE} || run_kubectl create namespace ${NAMESPACE}
            # apply manifest (manifest path is in workspace)
            run_kubectl apply -n ${NAMESPACE} -f ${WORKSPACE}/${MANIFEST}
          '''
        }
      }
    }

    stage('4 - Rollout verification') {
      steps {
        withCredentials([file(credentialsId: env.KUBECONFIG_CRED, variable: 'KUBECONFIG_FILE')]) {
          sh '''
            set -eux

            run_kubectl() {
              if command -v kubectl >/dev/null 2>&1; then
                kubectl "$@"
              else
                docker run --rm -v "${KUBECONFIG_FILE}:/kubeconfig:ro" -v "${WORKSPACE}:${WORKSPACE}" ${KUBECTL_DOCKER_IMAGE} \
                  kubectl --kubeconfig /kubeconfig "$@"
              fi
            }

            export KUBECONFIG="${KUBECONFIG_FILE}"
            run_kubectl rollout status deployment/${IMAGE_NAME}-http -n ${NAMESPACE} --timeout=120s
          '''
        }
      }
    }

    stage('5 - Pod validation using kubectl get pods') {
      steps {
        withCredentials([file(credentialsId: env.KUBECONFIG_CRED, variable: 'KUBECONFIG_FILE')]) {
          sh '''
            set -eux

            run_kubectl() {
              if command -v kubectl >/dev/null 2>&1; then
                kubectl "$@"
              else
                docker run --rm -v "${KUBECONFIG_FILE}:/kubeconfig:ro" -v "${WORKSPACE}:${WORKSPACE}" ${KUBECTL_DOCKER_IMAGE} \
                  kubectl --kubeconfig /kubeconfig "$@"
              fi
            }

            export KUBECONFIG="${KUBECONFIG_FILE}"
            echo "Pods in namespace ${NAMESPACE}:"
            run_kubectl get pods -n ${NAMESPACE} -o wide
          '''
        }
      }
    }

    stage('6 - Run a basic post-deploy test using curl (in-cluster)') {
      steps {
        withCredentials([file(credentialsId: env.KUBECONFIG_CRED, variable: 'KUBECONFIG_FILE')]) {
          sh '''
            set -eux

            run_kubectl() {
              if command -v kubectl >/dev/null 2>&1; then
                kubectl "$@"
              else
                docker run --rm -v "${KUBECONFIG_FILE}:/kubeconfig:ro" -v "${WORKSPACE}:${WORKSPACE}" ${KUBECTL_DOCKER_IMAGE} \
                  kubectl --kubeconfig /kubeconfig "$@"
              fi
            }

            export KUBECONFIG="${KUBECONFIG_FILE}"
            PODNAME="curl-test-$(date +%s)"

            # Use kubectl (local or dockerized) to run an ephemeral curl pod inside the cluster.
            # Note: inner command is single-quoted to preserve variable expansion inside the container.
            run_kubectl run "${PODNAME}" --rm -i --restart=Never --image=curlimages/curl -n ${NAMESPACE} --command -- sh -c '
              set -eux
              for i in 1 2 3 4 5; do
                echo "Attempt ${i}: curl -sS -m 5 http://${IMAGE_NAME}-http-svc/"
                if curl -sS -m 5 http://${IMAGE_NAME}-http-svc/; then
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

    stage('7 - Cleanup using kubectl delete -f <manifest>') {
      steps {
        withCredentials([file(credentialsId: env.KUBECONFIG_CRED, variable: 'KUBECONFIG_FILE')]) {
          sh '''
            set -eux || true

            run_kubectl() {
              if command -v kubectl >/dev/null 2>&1; then
                kubectl "$@"
              else
                docker run --rm -v "${KUBECONFIG_FILE}:/kubeconfig:ro" -v "${WORKSPACE}:${WORKSPACE}" ${KUBECTL_DOCKER_IMAGE} \
                  kubectl --kubeconfig /kubeconfig "$@"
              fi
            }

            export KUBECONFIG="${KUBECONFIG_FILE}"
            run_kubectl delete -n ${NAMESPACE} -f ${WORKSPACE}/${MANIFEST} --ignore-not-found=true || true
            echo "Cleanup completed."
          '''
        }
      }
    }
  } // stages

  post {
    always {
      archiveArtifacts artifacts: "${MANIFEST}", fingerprint: true, allowEmptyArchive: true
      echo 'Pipeline run finished.'
    }
    failure {
      echo 'Pipeline failed — check console output.'
    }
  }
}

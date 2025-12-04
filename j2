node('test-jenkins-cluster') {

    def REGISTRY = "nginx:alpine"
    def MANIFEST_FILE = "deployment.yaml"
    def KUBE_CRED_ID = "kubeconfig-file-test-cluster"
    def namespace = "test-ns"

    // --- Jenkins env vars if shell needs them 
    env.DOCKER_HOST = "tcp://dind:2375"
    env.DOCKER_TLS_VERIFY = "0"

    // --- Stage 1: Install kubectl using jnlp 
    stage('Install kubectl using jnlp') {
        container('jnlp') {
             echo "Installing kubectl..."
             sh '''
             # Install to user's local bin instead
             mkdir -p $HOME/bin
             curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
             chmod +x kubectl
             mv kubectl $HOME/bin/
             export PATH=$HOME/bin:$PATH
             kubectl version --client
            '''
        }
    }

    // --- Stage 2: Pull image from docker registry using dind container
    stage('Pull Image from Registry') {
        container('dind') {
            echo "Pulling Docker image using DinD..."
            sh "docker pull ${REGISTRY}"
        }
    }

    // --- Stage 3: Generate manifest at runtime of pipeline script
    stage('Generate Manifest at Runtime') {
        container('jnlp') {
            echo "Creating deployment manifest..."
            writeFile file: "${MANIFEST_FILE}", text: """
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-pod
  namespace: ${namespace}
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-pod
  template:
    metadata:
      labels:
        app: nginx-pod
    spec:
      containers:
      - name: nginx-pod
        image: ${REGISTRY}
        ports:
        - containerPort: 80
"""
        }
    }

    // --- Stage 4: Apply manifest with kubeconfig from Jenkins credentials ✅
    stage('Create a deplyment') {
        container('jnlp') {
            echo "Deploying to Kubernetes..."
            withCredentials([file(credentialsId: "${KUBE_CRED_ID}", variable: 'KUBE_CFG')]) {
                sh """
                export PATH=\$HOME/bin:\$PATH
                kubectl --kubeconfig=\$KUBE_CFG apply -f ${MANIFEST_FILE}
                """
            }
        }
    }

    // --- Stage 5: Rollout deployment 
    stage('Rollout deployment') {
        container('jnlp') {
            withCredentials([file(credentialsId: "${KUBE_CRED_ID}", variable: 'KUBE_CFG')]) {
                sh """
                export PATH=\$HOME/bin:\$PATH
                kubectl --kubeconfig=\$KUBE_CFG rollout status deployment/nginx-pod -n ${namespace}
                """
            }
        }
    }

    // --- Stage 6: check pod is running or not
    stage('check running pod') {
        container('jnlp') {
            withCredentials([file(credentialsId: "${KUBE_CRED_ID}", variable: 'KUBE_CFG')]) {
                 sh """
                export PATH=\$HOME/bin:\$PATH
                kubectl --kubeconfig=\$KUBE_CFG get pods -n ${namespace}
                """
            }
        }
    }

    // --- Stage 8: Cleanup all resources
    stage('Cleanup all resources') {
        container('jnlp') {
            echo "Deleting deployment..."
            withCredentials([file(credentialsId: "${KUBE_CRED_ID}", variable: 'KUBE_CFG')]) {
                sh """
                export PATH=\$HOME/bin:\$PATH
                kubectl --kubeconfig=\$KUBE_CFG delete -f ${MANIFEST_FILE} -n ${namespace}
                """
            }
        }
    }

    echo "All steps in pipeline are successfully executed and Pipeline is completed successfully"
}

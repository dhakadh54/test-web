node('test-jenkins-cluster') {

    def REGISTRY = "nginx:alpine"
    def MANIFEST_FILE = "deployment.yaml"
    def KUBE_CRED_ID = "kubeconfig-file-test-cluster"
    def namespace = "test-ns"

    // --- Jenkins env vars if shell needs them 
    env.DOCKER_HOST = "tcp://dind:2375"
    env.DOCKER_TLS_VERIFY = "0"

    // --- Stage 1: Install kubectl 
    stage('Install kubectl') {
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

    // --- Stage 2: Pull image using DIND container ✅
    stage('Pull Image from Registry') {
        container('dind') {
            echo "Pulling Docker image using DinD..."
            sh "docker pull ${REGISTRY}"
        }
    }

    // --- Stage 3: Generate manifest at runtime ✅
    stage('Generate Manifest at Runtime') {
        container('jnlp') {
            echo "Creating deployment manifest..."
            writeFile file: "${MANIFEST_FILE}", text: """
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test-nginx
  namespace: ${namespace}
spec:
  replicas: 1
  selector:
    matchLabels:
      app: test-nginx
  template:
    metadata:
      labels:
        app: test-nginx
    spec:
      containers:
      - name: test-nginx
        image: ${REGISTRY}
        ports:
        - containerPort: 80
"""
        }
    }

    // --- Stage 4: Apply manifest with kubeconfig from Jenkins credentials ✅
    stage('kubectl Apply') {
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

    // --- Stage 5: Rollout check ✅
    stage('Rollout Verification') {
        container('jnlp') {
            withCredentials([file(credentialsId: "${KUBE_CRED_ID}", variable: 'KUBE_CFG')]) {
                sh """
                export PATH=\$HOME/bin:\$PATH
                kubectl --kubeconfig=\$KUBE_CFG rollout status deployment/test-nginx -n ${namespace}
                """
            }
        }
    }

    // --- Stage 6: Pod status ✅
    stage('Pod Validation') {
        container('jnlp') {
            withCredentials([file(credentialsId: "${KUBE_CRED_ID}", variable: 'KUBE_CFG')]) {
                 sh """
                export PATH=\$HOME/bin:\$PATH
                kubectl --kubeconfig=\$KUBE_CFG get pods -n ${namespace}
                """
            }
        }
    }

    // --- Stage 8: Cleanup ✅
    stage('Cleanup') {
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

    echo "✅ Pipeline completed"
}

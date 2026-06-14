pipeline {
    agent any

    // Uses the native Jenkins Maven installation configured in Global Tool Management
    tools {
        maven 'maven-3.9.6'
    }

    environment {
        CLUSTER_URL  = 'https://192.168.1.6:6443'
        APP_NAME     = 'spring-boot-app'
        NAMESPACE    = 'jenkins-apps'
        IMAGE_TAG    = "${APP_NAME}:${BUILD_NUMBER}"
    }

    stages {
        stage('1. Maven Build & Test') {
            steps {
                echo 'Compiling App using Native Maven Tool...'
                sh 'mvn clean package -DskipTests'
            }
        }

        stage('2. Build Docker Image') {
            steps {
                echo "Installing Docker CLI binary and building image..."
                sh """
                    # 1. Download official static Docker CLI binary if missing
                    if [ ! -f ./docker ]; then
                        echo "Downloading Docker CLI v26.1.4..."
                        curl -fsSL https://download.docker.com/linux/static/stable/x86_64/docker-26.1.4.tgz | tar -xzO docker/docker > ./docker
                        chmod +x ./docker
                    fi
                    
                    # 2. Run the build using our local downloaded binary
                    ./docker build -t ${IMAGE_TAG} .

                    # 3. Transfer the image from Docker over into the Local Kubernetes Containerd Cache
                    echo "Syncing image into containerd k8s namespace..."
                    ./docker save ${IMAGE_TAG} | ctr -n k8s.io images import -
                """
            }
        }

        stage('3. Push NodePort Deployments to Rancher') {
            steps {
                echo "Interacting with Rancher API..."
                withCredentials([string(credentialsId: 'kubernetes-cluster-token', variable: 'KUBE_TOKEN')]) {
                    sh """
                        # Create or Update Kubernetes Deployment
                        curl -k -X POST -H "Authorization: Bearer \${KUBE_TOKEN}" -H "Content-Type: application/yaml" --data "
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ${APP_NAME}
  namespace: ${NAMESPACE}
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ${APP_NAME}
  template:
    metadata:
      labels:
        app: ${APP_NAME}
    spec:
      containers:
      - name: spring-app
        image: ${IMAGE_TAG}
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 8080
" ${CLUSTER_URL}/apis/apps/v1/namespaces/${NAMESPACE}/deployments || echo 'Deployment already exists'

                        # Create or Update NodePort Service
                        curl -k -X POST -H "Authorization: Bearer \${KUBE_TOKEN}" -H "Content-Type: application/yaml" --data "
apiVersion: v1
kind: Service
metadata:
  name: ${APP_NAME}-service
  namespace: ${NAMESPACE}
spec:
  type: NodePort
  selector:
    app: ${APP_NAME}
  ports:
  - port: 8080
    targetPort: 8080
    nodePort: 32085
" ${CLUSTER_URL}/api/v1/namespaces/${NAMESPACE}/services || echo 'Service already exists'
                    """
                }
            }
        }
    }
    
    post {
        always {
            cleanWs()
        }
    }
}

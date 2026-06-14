pipeline {
    agent any

    tools {
        maven 'maven-3.9.6'
    }

    environment {
        CLUSTER_URL  = 'https://192.168.1.6:6443'
        APP_NAME     = 'spring-boot-app'
        NAMESPACE    = 'jenkins-apps'
        
        // Matches your Docker Hub account username
        DOCKER_USER  = 'indu1999'
        IMAGE_TAG    = "${DOCKER_USER}/${APP_NAME}:${BUILD_NUMBER}"
        
        // Custom variables for your environment links
        REPO_URL     = 'https://github.com/Induwara1999/spring-boot-app' // Update if your repo name is different
        JENKINS_URL  = 'http://192.168.1.6:8080' // Standard local Jenkins server URL setup
    }

    stages {
        stage('1. Maven Build & Test') {
            steps {
                echo 'Compiling App using Native Maven Tool...'
                sh 'mvn clean package -DskipTests'
            }
        }

        stage('2. Build & Push Docker Image') {
            steps {
                echo "Installing Docker CLI binary and managing image lifecycle..."
                withCredentials([usernamePassword(credentialsId: 'docker-hub-credentials', usernameVariable: 'HUB_USER', passwordVariable: 'HUB_TOKEN')]) {
                    sh """
                        # Download official static Docker CLI binary if missing
                        if [ ! -f ./docker ]; then
                            echo "Downloading Docker CLI v26.1.4..."
                            curl -fsSL https://download.docker.com/linux/static/stable/x86_64/docker-26.1.4.tgz | tar -xzO docker/docker > ./docker
                            chmod +x ./docker
                        fi
                        
                        # Build the image using the environment tags
                        ./docker build -t ${IMAGE_TAG} .
                        
                        # Clean stream pipe login forcing the verified token variable value
                        echo "\$HUB_TOKEN" | ./docker login --username "${DOCKER_USER}" --password-stdin
                        
                        # Push the image layers to Docker Hub
                        ./docker push ${IMAGE_TAG}
                    """
                }
            }
        }

        stage('3. Push NodePort Deployments to Rancher') {
            steps {
                echo "Interacting with Rancher API..."
                withCredentials([string(credentialsId: 'kubernetes-cluster-token', variable: 'KUBE_TOKEN')]) {
                    sh """
                        # Create or Update Kubernetes Deployment with custom descriptive annotations
                        curl -k -X POST -H "Authorization: Bearer \${KUBE_TOKEN}" -H "Content-Type: application/yaml" --data "
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ${APP_NAME}
  namespace: ${NAMESPACE}
  annotations:
    field.cattle.io/description: 'Source Code: ${REPO_URL} | Build Pipeline: ${JENKINS_URL}'
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
        imagePullPolicy: Always
        ports:
        - containerPort: 8080
" ${CLUSTER_URL}/apis/apps/v1/namespaces/${NAMESPACE}/deployments || curl -k -X PATCH -H "Authorization: Bearer \${KUBE_TOKEN}" -H "Content-Type: application/strategic-merge-patch+json" --data "{\\"spec\\":{\\"template\\":{\\"spec\\":{\\"containers\\":[{\\"name\\":\\"spring-app\\",\\"image\\":\\"${IMAGE_TAG}\\"}]}}}}" ${CLUSTER_URL}/apis/apps/v1/namespaces/${NAMESPACE}/deployments/${APP_NAME}

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

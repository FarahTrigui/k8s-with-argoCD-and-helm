pipeline {
    agent any
    
    tools {
        maven 'Maven_3.8.1'
        jdk 'jdk17'
    }
    
    environment {
        // Application info
        APP_NAME = 'springboot-app'
        VERSION = "${env.BUILD_NUMBER}"
        
        // Docker registry info
        DOCKER_REGISTRY = 'docker.io'
        DOCKER_REPO = 'fifia/springboot-app'
        DOCKER_CREDS = 'docker'
        
        // Kubernetes config
        KUBECONFIG = '/var/lib/jenkins/.kube/config'
        
        // Helm chart path
        HELM_CHART_PATH = './'
        
        // SonarQube configuration
        SONAR_SERVER = 'sonar-server'
        
        // ArgoCD configuration
        ARGOCD_SERVER = 'localhost:8090'
    }
    
    stages {
        stage('Checkout') {
            steps {
                echo 'Checking out source code...'
                checkout scm
            }
        }
        
        stage('Build') {
            steps {
                echo 'Building the application with Maven...'
                sh 'cd spring-boot-app && mvn clean compile'
            }
        }
        
        stage('Unit Tests') {
            steps {
                echo 'Running unit tests...'
                sh ' cd spring-boot-app && mvn test -B || true '
            }
        }
        
        stage('SonarQube Analysis') {
            steps {
                echo 'Running SonarQube analysis...'
                withSonarQubeEnv(SONAR_SERVER) {
                    sh 'cd spring-boot-app && mvn sonar:sonar'
                }
                timeout(time: 10, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }
        
        stage('Package') {
            steps {
                echo 'Packaging the application...'
                sh 'cd spring-boot-app && mvn package -DskipTests'
            }
        }
        
        stage('Build and Push Docker Image') {
            steps {
                echo 'Building and pushing Docker image...'
                withCredentials([usernamePassword(credentialsId: DOCKER_CREDS, passwordVariable: 'DOCKER_PASSWORD', usernameVariable: 'DOCKER_USERNAME')]) {
                    sh '''
                       cd spring-boot-app &&  docker build -t ${DOCKER_REGISTRY}/${DOCKER_REPO}:${VERSION} .
                        docker tag ${DOCKER_REGISTRY}/${DOCKER_REPO}:${VERSION} ${DOCKER_REGISTRY}/${DOCKER_REPO}:latest
                        echo ${DOCKER_PASSWORD} | docker login ${DOCKER_REGISTRY} -u ${DOCKER_USERNAME} --password-stdin
                        docker push ${DOCKER_REGISTRY}/${DOCKER_REPO}:${VERSION}
                        docker push ${DOCKER_REGISTRY}/${DOCKER_REPO}:latest
                    '''
                }
            }
        }
        
        stage('Deploy to Test Environment') {
            steps {
                echo 'Deploying to test environment using Helm...'
                sh '''
                    export KUBECONFIG=${KUBECONFIG}
                    
                    # Update the image tag in values.yaml
                    sed -i "s|tag:.*|tag: ${VERSION}|g" ${HELM_CHART_PATH}/values.yaml
                    
                    # Deploy or upgrade the Helm release in test namespace
                    helm upgrade --install ${APP_NAME}-test ${HELM_CHART_PATH} \
                        --namespace test --create-namespace \
                        --set image.repository=${DOCKER_REGISTRY}/${DOCKER_REPO} \
                        --set image.tag=${VERSION} \
                        --wait --timeout 5m
                    
                    # Get the service URL to use for testing
                    export SERVICE_URL=$(kubectl get svc -n test ${APP_NAME}-test -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
                    echo "Application deployed at: http://${SERVICE_URL}"
                '''
            }
        }
        
       stage('Bump Chart Version') {
        steps {
    withCredentials([usernamePassword(credentialsId: 'github', usernameVariable: 'GIT_USER', passwordVariable: 'GIT_TOKEN')]) {
      sh '''
        git config user.email "farahtrigui5@gmail.com"
        git config user.name  "FarahTrigui"

        # Replace the tag
        sed -i "s/tag: \\".*\\"/tag: \\"${BUILD_NUMBER}\\"/g" values.yaml

        # Git commit & push using credentials
        git add values.yaml
        git commit -m "ci: bump image.tag to ${BUILD_NUMBER}"

        git push https://${GIT_USER}:${GIT_TOKEN}@github.com/${GIT_USER}/k8s-with-argoCD-and-helm.git HEAD:main
      '''
    }
  }
}
                   
    }
    
    post {
        always {
            echo 'Cleaning up workspace...'
            cleanWs()
        }
        success {
            echo 'CI/CD pipeline completed successfully!'
            // You could add notifications here (email, Slack, etc.)
        }
        failure {
            echo 'CI/CD pipeline failed!'
            // You could add failure notifications here
        }
    }
}

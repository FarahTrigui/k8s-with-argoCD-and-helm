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
        KUBECONFIG = credentials('kubernetes')
        
        // Helm chart path
        HELM_CHART_PATH = './springboot-app'
        
        // SonarQube configuration
        SONAR_SERVER = 'sonar-token'
        
        // ArgoCD configuration
        ARGOCD_SERVER = 'localhost:8090'
        ARGOCD_CREDS = 'argocd-credentials-id'
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
            post {
                always {
                    junit '**/target/surefire-reports/*.xml'
                }
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
                archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
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
        
        stage('User Acceptance Tests') {
            steps {
                echo 'Running UAT on deployed application...'
                sh '''
                    export KUBECONFIG=${KUBECONFIG}
                    
                    # Get the service URL
                    export SERVICE_URL=$(kubectl get svc -n test ${APP_NAME}-test -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
                    
                    # Wait for the application to be ready
                    for i in {1..30}; do
                        statusCode=$(curl -s -o /dev/null -w "%{http_code}" http://${SERVICE_URL}/actuator/health || echo "000")
                        if [ "$statusCode" = "200" ]; then
                            echo "Application is up and running"
                            break
                        fi
                        echo "Waiting for application to be ready... ($i/30)"
                        sleep 10
                    done
                    
                    # Run your UAT tests
                    # This could be a separate test suite or API tests
                    mvn verify -Pintegration-test -Dtest.host=${SERVICE_URL}
                '''
            }
        }
        
        stage('Promote to Production with ArgoCD') {
            steps {
                echo 'Promoting application to production using ArgoCD...'
                withCredentials([usernamePassword(credentialsId: ARGOCD_CREDS, passwordVariable: 'ARGOCD_PASSWORD', usernameVariable: 'ARGOCD_USERNAME')]) {
                    sh '''
                        # Install ArgoCD CLI if not available
                        if ! command -v argocd &> /dev/null; then
                            curl -sSL -o /usr/local/bin/argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
                            chmod +x /usr/local/bin/argocd
                        fi
                        
                        # Update the Git repository with the new version
                        git config --global user.email "jenkins@example.com"
                        git config --global user.name "Jenkins Pipeline"
                        
                        # Update the image tag in values-prod.yaml or create a new version-specific values file
                        sed -i "s|tag:.*|tag: ${VERSION}|g" ${HELM_CHART_PATH}/values-prod.yaml
                        
                        # Commit and push changes
                        git add ${HELM_CHART_PATH}/values-prod.yaml
                        git commit -m "Update application version to ${VERSION} for production deployment"
                        git push origin HEAD:main
                        
                        # Login to ArgoCD
                        argocd login ${ARGOCD_SERVER} --username ${ARGOCD_USERNAME} --password ${ARGOCD_PASSWORD} --insecure
                        
                        # Sync the ArgoCD application
                        argocd app sync ${APP_NAME}-prod
                        
                        # Wait for the sync to complete
                        argocd app wait ${APP_NAME}-prod --health --timeout 300
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

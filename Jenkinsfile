pipeline {
    agent any

    tools {
        jdk 'Java_17'
        maven 'maven_3.6.3'
    }

    environment {
        DOCKERHUB_USER = "ram1993"
        DOCKERHUB_REPO = "java-aks-app"
        IMAGE_TAG = "${BUILD_NUMBER}"
        IMAGE_NAME = "${DOCKERHUB_USER}/${DOCKERHUB_REPO}"

        DOCKERHUB_CRED = credentials('dockerhub-cred')
        KUBECONFIG_CRED = credentials('aks-kubeconfig')
    }

    stages {

        stage("Checkout Code") {
            steps {
                git branch: 'main',
                url: 'https://github.com/ramanji1990/poc-jenkins-cicd.git'
            }
        }

        stage("Maven Build") {
            steps {
                sh '''
                java -version
                mvn -version
                mvn clean package
                '''
            }
        }

        stage("SonarQube Code Scan") {
            steps {
                withSonarQubeEnv('sonar') {
                    sh '''
                    mvn sonar:sonar \
                      -Dsonar.projectKey=java-aks-app \
                      -Dsonar.projectName=java-aks-app
                    '''
                }
            }
        }

        stage("SonarQube Quality Gate") {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage("Build Docker Image") {
            steps {
                sh '''
                docker build -t ${IMAGE_NAME}:${IMAGE_TAG} .
                docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${IMAGE_NAME}:latest
                '''
            }
        }

        stage("Trivy HTML Report") {
            steps {
                sh '''
                mkdir -p trivy-report
                trivy image \
                  --format template \
                  --template "@/usr/local/share/trivy/templates/html.tpl" \
                  --output trivy-report/trivy-report.html \
                  ${IMAGE_NAME}:${IMAGE_TAG}
                '''
            }
        }

        stage("Publish Trivy Report") {
            steps {
                publishHTML([
                    allowMissing: false,
                    alwaysLinkToLastBuild: true,
                    keepAll: true,
                    reportDir: 'trivy-report',
                    reportFiles: 'trivy-report.html',
                    reportName: 'Trivy Security Report'
                ])
            }
        }

        stage("Trivy Security Gate") {
            steps {
                sh '''
                trivy image \
                  --exit-code 1 \
                  --severity CRITICAL \
                  ${IMAGE_NAME}:${IMAGE_TAG}
                '''
            }
        }

        stage("DockerHub Login") {
            steps {
                sh '''
                echo ${DOCKERHUB_CRED_PSW} | docker login \
                  -u ${DOCKERHUB_CRED_USR} \
                  --password-stdin
                '''
            }
        }

        stage("Push Image to DockerHub") {
            steps {
                sh '''
                docker push ${IMAGE_NAME}:${IMAGE_TAG}
                docker push ${IMAGE_NAME}:latest
                '''
            }
        }

        stage("Deploy to AKS") {
            steps {
                sh '''
                mkdir -p ~/.kube
                cp "$KUBECONFIG_CRED" ~/.kube/config
                chmod 600 ~/.kube/config

                kubectl apply -f k8s/deployment.yaml
                kubectl apply -f k8s/service.yaml

                kubectl set image deployment/java-aks-app \
                  java-aks-app='"${IMAGE_NAME}:${IMAGE_TAG}"'

                kubectl rollout status deployment/java-aks-app --timeout=120s || true

                kubectl get pods
                kubectl get svc
                '''
            }
        }
    }

    post {
        always {
            echo "Pipeline execution completed"
        }

        success {
            echo "CI/CD + DevSecOps pipeline completed successfully 🚀"
        }

        failure {
            echo "Pipeline failed. Check logs"
        }
    }
}
pipeline {
    agent { label 'Jenkins-Agent' }

    tools {
        jdk 'Java17'
        maven 'Maven3'
    }

    environment {
        APP_NAME = "register-app-pipeline"
        RELEASE  = "1.0.0"
        DOCKER_USER = "ashfaque9x"
        IMAGE_NAME  = "${DOCKER_USER}/${APP_NAME}"
        IMAGE_TAG   = "${RELEASE}-${BUILD_NUMBER}"

        JENKINS_API_TOKEN = credentials("JENKINS_API_TOKEN")
    }

    stages {

        stage("Cleanup Workspace") {
            steps {
                cleanWs()
            }
        }

        stage("Checkout from SCM") {
            steps {
                git branch: 'main',
                    credentialsId: 'github',
                    url: 'https://github.com/Ashfaque-9x/register-app'
            }
        }

        stage("Build Application") {
            steps {
                sh "mvn clean package"
            }
        }

        stage("Test Application") {
            steps {
                sh "mvn test"
            }
        }

        /* SonarQube stages commented out until server is ready */

        stage("Build & Push Docker Image") {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', 
                                                 usernameVariable: 'DOCKER_USER', 
                                                 passwordVariable: 'DOCKER_PASS')]) {
                    script {
                        // Login to Docker Hub
                        sh "docker login -u $DOCKER_USER -p $DOCKER_PASS"

                        // Build Docker image
                        sh "docker build -t ${IMAGE_NAME}:latest ."
                        sh "docker tag ${IMAGE_NAME}:latest ${IMAGE_NAME}:${IMAGE_TAG}"

                        // Push Docker image
                        sh "docker push ${IMAGE_NAME}:latest"
                        sh "docker push ${IMAGE_NAME}:${IMAGE_TAG}"
                    }
                }
            }
        }

        stage("Trivy Scan") {
            steps {
                sh """
                docker run --rm \
                -v /var/run/docker.sock:/var/run/docker.sock \
                aquasec/trivy image ${IMAGE_NAME}:${IMAGE_TAG} \
                --no-progress --severity HIGH,CRITICAL --exit-code 0
                """
            }
        }

        stage("Cleanup Artifacts") {
            steps {
                sh "docker rmi ${IMAGE_NAME}:${IMAGE_TAG} || true"
                sh "docker rmi ${IMAGE_NAME}:latest || true"
            }
        }

        stage("Trigger CD Pipeline") {
            steps {
                sh """
                curl -X POST \
                --user clouduser:${JENKINS_API_TOKEN} \
                http://ec2-13-232-128-192.ap-south-1.compute.amazonaws.com:8080/job/gitops-register-app-cd/buildWithParameters \
                --data IMAGE_TAG=${IMAGE_TAG} \
                --data token=gitops-token
                """
            }
        }
    }

    post {
        failure {
            echo "Build failed"
        }
    }
}

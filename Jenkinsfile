pipeline {
    agent any

    environment {
        IMAGE_NAME = "myapi-img"
        CONTAINER_NAME = "myapi-container"
        NETWORK_NAME = "jenkins-net"
        API_PORT = "8290"
        MANAGEMENT_PORT = "8253"
    }

    stages {

        stage ('Prepare'){
            steps{
                echo 'Workspace ready: Jenkins will clone repository automatically'
            }
        }

        stage('Build Docker Image') {
            steps {
                echo "🔧 Building Docker image"
                sh "docker build -t ${IMAGE_NAME}:v1 ."
            }
        }

        stage('Stop & Remove Old Container') {
            steps {
                echo "Stopping old container (if exists)"
                sh """
                    docker stop ${CONTAINER_NAME} || true
                    docker rm ${CONTAINER_NAME} || true
                """
            }
        }

        stage('Run New Container') {
            steps {
                echo "🚀 Running new container..."
                sh """
                    docker run -d \
                    --name ${CONTAINER_NAME} \
                    --network ${NETWORK_NAME} \
                    -p ${API_PORT}:${API_PORT} \
                    -p ${MANAGEMENT_PORT}:${MANAGEMENT_PORT} \
                    ${IMAGE_NAME}:v1
                """
            }
        }


        stage('Test API') {
            steps {
                echo "Wait 10 seconds for WSO2 MI to fully start"
                sh """
                    sleep 10
                    curl -I http://${CONTAINER_NAME}:8290 || true
                """
            }
        }
    }

    post {
        always {
            echo "✅ Pipeline finished!"
        }
    }
}

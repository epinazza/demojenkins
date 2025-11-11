pipeline {
    agent any
    environment {
        IMAGE_NAME = "myapi-img"
        JMETER_IMAGE = "my-jmeter-img:latest"
        CONTAINER_NAME = "myapi-container"
        JMETER_CONTAINER = "jmeter-agent"
        NETWORK_NAME = "jenkins-net"
        API_PORT = "8290"
        JMX_FILE = "API_TestPlan.jmx"
        RESULTS_DIR = "results"
    }
    stages {
        stage('Prepare') {
            steps {
                echo 'Workspace ready: repository cloned'
            }
        }
        stage('Build Docker Image') {
            steps {
                echo "Building Docker image ${IMAGE_NAME}:v1"
                sh "docker build -t ${IMAGE_NAME}:v1 ."
            }
        }
        stage('Stop & Remove Old Container') {
            steps {
                echo "Stopping and removing old container if exists"
                sh """
                    docker stop ${CONTAINER_NAME} || true
                    docker rm ${CONTAINER_NAME} || true
                """
            }
        }
        stage('Run Docker Container') {
            steps {
                echo "Running new container ${CONTAINER_NAME}"
                sh """
                    docker run -d \
                    --name ${CONTAINER_NAME} \
                    --network ${NETWORK_NAME} \
                    -p ${API_PORT}:${API_PORT} \
                    ${IMAGE_NAME}:v1
                """
            }
        }
        stage('Verify Container') {
            steps {
                echo 'Listing running containers'
                sh "docker ps"
            }
        }
        stage('Test APIs') {
            steps {
                script {
                    def apis = [
                        [method: 'GET', path: '/appointmentservices/getAppointment'],
                        [method: 'PUT', path: '/appointmentservices/setAppointment']
                    ]
                    apis.each { api ->
                        echo "Waiting for ${api.method} ${api.path}..."
                        def ready = false
                        for (int i = 1; i <= 12; i++) { // Try 12 times, 10s apart
                            sleep 10
                            def status = sh(
                                script: "curl -o /dev/null -s -w '%{http_code}' -X ${api.method} http://${CONTAINER_NAME}:${API_PORT}${api.path}",
                                returnStdout: true
                            ).trim()
                            echo "Attempt ${i}: HTTP ${status}"
                            if (status == "200" || status == "202") {
                                ready = true
                                echo "${api.method} ${api.path} is ready!"
                                break
                            }
                        }
                        if (!ready) {
                            error "${api.method} ${api.path} not ready after 2 minutes"
                        }
                    }
                }
            }
        }
        stage('Run JMeter Load Test') {
            steps {
                echo "🏃 Running JMeter load test..."
                sh '''
                    # Auto-detect JMX file in workspace
                    JMX_PATH=$(find /workspace -type f -name "${JMX_FILE}" | head -n 1)

                    if [ -z "$JMX_PATH" ]; then
                        echo "❌ Could not find ${JMX_FILE} inside container path"
                        exit 1
                    fi

                    echo "✅ Found JMX file at: $JMX_PATH"
                    echo "🧪 Running JMeter test inside Docker..."

                    docker run --rm --name ${JMETER_CONTAINER} \
                        --network ${NETWORK} \
                        -v /var/jenkins_home/workspace/pipelineA:/workspace \
                        -v /var/jenkins_home/workspace/pipelineA/results:/results \
                        -w /workspace \
                        ${JMETER_IMAGE} \
                        -n -t "$JMX_PATH" -l /results/report.jtl
                '''
            }
        }

    }
    post {
        always {
            echo "✅ Pipeline finished!"
        }
    }
}
 
 
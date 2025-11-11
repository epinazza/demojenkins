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
        stage('Clean Workspace') {
            steps {
                echo "🧹 Cleaning workspace..."
                deleteDir()
            }
        }

        stage('Checkout SCM') {
            steps {
                checkout([$class: 'GitSCM',
                    branches: [[name: '*/main']],
                    userRemoteConfigs: [[url: 'https://github.com/epinazza/demojenkins.git']]
                ])
            }
        }

        stage('Prepare') {
            steps {
                echo 'Workspace ready: repository cloned'
            }
        }

        stage('Prepare Workspace') {
            steps {
                echo "📂 Preparing workspace directories..."
                sh "mkdir -p ${RESULTS_DIR}"
                sh "rm -rf ${RESULTS_DIR}/* || true"
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
                    docker stop ${JMETER_CONTAINER} || true
                    docker rm ${JMETER_CONTAINER} || true
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
                        for (int i = 1; i <= 12; i++) { // Try 12 times, 5s apart
                            sleep 5
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
                sh """
                    if [ ! -f "${WORKSPACE}/${JMX_FILE}" ]; then
                        echo "❌ JMX file not found in workspace: ${WORKSPACE}/${JMX_FILE}"
                        exit 1
                    fi

                    docker run --rm --name ${JMETER_CONTAINER} \
                        --network ${NETWORK_NAME} \
                        -v ${WORKSPACE}:/workspace \
                        -v ${WORKSPACE}/${RESULTS_DIR}:/results \
                        -w /workspace \
                        ${JMETER_IMAGE} \
                        -n -t /workspace/${JMX_FILE} -l /results/report.jtl
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
 
 
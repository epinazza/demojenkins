pipeline {
    agent any

    environment {
        WORKSPACE_DIR = "${env.WORKSPACE}"
        DOCKER_NET = "jenkins-net"
        API_CONTAINER = "myapi-container"
        IMAGE_NAME = "myapi-img:v1"
        JMETER_IMAGE = "justb4/jmeter:latest"
        JMX_FILE = "API_TestPlan.jmx"
        RESULT_DIR = "results"
    }

    stages {
        stage('Prepare Workspace') {
            steps {
                echo "🛠 Workspace ready"
                sh "mkdir -p ${WORKSPACE_DIR}/${RESULT_DIR}"
            }
        }

        stage('Build Docker Image') {
            steps {
                echo "🔧 Building Docker image..."
                sh "docker build -t ${IMAGE_NAME} ."
            }
        }

        stage('Stop & Remove Old Container') {
            steps {
                echo "🧹 Cleaning up old WSO2 container..."
                sh """
                    docker stop ${API_CONTAINER} || true
                    docker rm ${API_CONTAINER} || true
                """
            }
        }

        stage('Run WSO2 Container') {
            steps {
                echo "🚀 Starting WSO2 Micro Integrator container..."
                sh """
                    docker network create ${DOCKER_NET} || true
                    docker run -d --name ${API_CONTAINER} --network ${DOCKER_NET} -p 8290:8290 -p 8253:8253 ${IMAGE_NAME}
                """
            }
        }

        stage('Wait for API Ready') {
            steps {
                echo "⏳ Waiting 40 seconds for API to be ready..."
                sh "sleep 40"
                script {
                    def status = sh(script: "curl -s -o /dev/null -w %{http_code} http://host.docker.internal:8290/appointmentservices/getAppointment", returnStdout: true).trim()
                    echo "HTTP status: ${status}"
                }
            }
        }

        stage('Run JMeter Load Test') {
            steps {
                echo "🏃 Running JMeter load test..."
                sh """
                    docker run --rm --name jmeter-agent \
                    --network ${DOCKER_NET} -u root \
                    -v ${WORKSPACE_DIR}:/tests -w /tests \
                    ${JMETER_IMAGE} \
                    -n -t ${JMX_FILE} -l ${RESULT_DIR}/report.jtl
                """
            }
        }

        stage('Archive JMeter Report') {
            steps {
                echo "📦 Archiving JMeter report..."
                archiveArtifacts artifacts: "${RESULT_DIR}/report.jtl", allowEmptyArchive: true
            }
        }
    }

    post {
        always {
            echo "🧹 Cleaning up Docker containers..."
            sh """
                docker stop ${API_CONTAINER} || true
                docker rm ${API_CONTAINER} || true
            """
        }
        failure {
            echo "❌ Pipeline failed!"
        }
    }
}

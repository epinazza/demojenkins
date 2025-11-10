pipeline {
    agent any
    environment {
        API_CONTAINER = "myapi-container"
        API_IMAGE     = "myapi-img:v1"
        JMETER_CONTAINER = "jmeter-agent"
        NETWORK_NAME = "jenkins-net"
        API_PORT = "8290"
    }
    stages {

        stage('Checkout SCM') {
            steps {
                checkout scm
            }
        }

        stage('Prepare Workspace') {
            steps {
                echo "🛠 Workspace ready"
                sh '''
                    mkdir -p ${WORKSPACE}/results
                '''
            }
        }

        stage('Build Docker Image') {
            steps {
                echo "🔧 Building Docker image..."
                sh "docker build -t ${API_IMAGE} ."
            }
        }

        stage('Stop & Remove Old API Container') {
            steps {
                echo "🧹 Cleaning up old WSO2 container (if any)..."
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
                    docker network create ${NETWORK_NAME} || true
                    docker run -d --name ${API_CONTAINER} --network ${NETWORK_NAME} -p ${API_PORT}:${API_PORT} -p 8253:8253 ${API_IMAGE}
                """
            }
        }

        stage('Wait for API Ready') {
            steps {
                echo "⏳ Waiting 40 seconds for API to be ready..."
                sh 'sleep 40'
                script {
                    def status = sh(
                        script: "curl -s -o /dev/null -w %{http_code} http://host.docker.internal:${API_PORT}/appointmentservices/getAppointment",
                        returnStdout: true
                    ).trim()
                    echo "HTTP status: ${status}"
                    if (status != "200") {
                        error("API not ready, HTTP ${status}")
                    }
                }
            }
        }

        stage('Check JMX File') {
            steps {
                echo "📄 Checking JMX file..."
                script {
                    def jmxExists = fileExists("${WORKSPACE}/API_TestPlan.jmx")
                    if (!jmxExists) {
                        error("JMX file not found in workspace!")
                    }
                }
            }
        }

        stage('Run Load Test with JMeter') {
            steps {
                echo "🏃 Running JMeter load test..."
                sh """
                    docker run --rm --name ${JMETER_CONTAINER} --network ${NETWORK_NAME} -u root \
                    -v ${WORKSPACE}:/tests -w /tests justb4/jmeter:latest \
                    -n -t API_TestPlan.jmx -l results/report.jtl
                """
            }
        }

        stage('Archive JMeter Report') {
            steps {
                archiveArtifacts artifacts: 'results/**', allowEmptyArchive: true
            }
        }
    }

    post {
        always {
            echo "🧹 Cleaning up Docker containers..."
            sh """
                docker stop ${API_CONTAINER} || true
                docker rm ${API_CONTAINER} || true
                docker stop ${JMETER_CONTAINER} || true
                docker rm ${JMETER_CONTAINER} || true
            """
        }
        success {
            echo "✅ Pipeline completed successfully!"
        }
        failure {
            echo "❌ Pipeline failed!"
        }
    }
}

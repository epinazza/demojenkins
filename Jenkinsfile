pipeline {
    agent any

    environment {
        WSO2_IMAGE = "myapi-img:v1"
        WSO2_CONTAINER = "myapi-container"
        JMETER_CONTAINER = "jmeter-agent"
        NETWORK_NAME = "jenkins-net"
        RESULTS_DIR = "${WORKSPACE}/results"
        JMX_FILE = "API_TestPlan.jmx" // Make sure this exists in your repo
    }

    stages {

        stage('Checkout SCM') {
            steps {
                checkout scm
            }
        }

        stage('Prepare') {
            steps {
                echo "🛠 Workspace ready"
                sh "mkdir -p ${RESULTS_DIR}"
            }
        }

        stage('Build Docker Image') {
            steps {
                echo "🔧 Building Docker image..."
                sh "docker build -t ${WSO2_IMAGE} ."
            }
        }

        stage('Stop & Remove Old Container') {
            steps {
                echo "🧹 Cleaning up old WSO2 container (if any)..."
                sh """
                    docker stop ${WSO2_CONTAINER} || true
                    docker rm ${WSO2_CONTAINER} || true
                """
            }
        }

        stage('Run WSO2 Container') {
            steps {
                echo "🚀 Starting WSO2 Micro Integrator container..."
                sh """
                    docker network create ${NETWORK_NAME} || true
                    docker run -d --name ${WSO2_CONTAINER} --network ${NETWORK_NAME} -p 8290:8290 -p 8253:8253 ${WSO2_IMAGE}
                """
            }
        }

        stage('Wait for API Ready') {
            steps {
                echo "⏳ Waiting 40 seconds for API to be ready..."
                sh "sleep 40"

                script {
                    def status = sh(
                        script: "curl -s -o /dev/null -w %{http_code} http://host.docker.internal:8290/appointmentservices/getAppointment",
                        returnStdout: true
                    ).trim()

                    echo "HTTP status: ${status}"
                    if (status != "200") {
                        error "API is not ready, HTTP status ${status}"
                    }
                }
            }
        }

        stage('Check JMX File') {
            steps {
                echo "📄 Checking JMX file..."
                script {
                    if (!fileExists("${JMX_FILE}")) {
                        error "JMX file not found!"
                    }
                }
            }
        }

        stage('Run Load Test with JMeter') {
            steps {
                echo "🏃 Running JMeter load test..."
                sh """
                    docker stop ${JMETER_CONTAINER} || true
                    docker rm ${JMETER_CONTAINER} || true
                    docker run -d --name ${JMETER_CONTAINER} \\
                        --network ${NETWORK_NAME} \\
                        -v ${WORKSPACE}:/tests \\
                        justb4/jmeter:latest \\
                        jmeter -n -t /tests/${JMX_FILE} -l /tests/results/report.jtl
                """

                // Wait for JMeter test to complete
                sh "docker wait ${JMETER_CONTAINER}"
            }
        }

        stage('Archive JMeter Report') {
            steps {
                echo "📦 Archiving JMeter report..."
                archiveArtifacts artifacts: 'results/*.jtl', allowEmptyArchive: true
            }
        }
    }

    post {
        always {
            echo "🧹 Cleaning up Docker containers..."
            sh """
                docker stop ${WSO2_CONTAINER} || true
                docker rm ${WSO2_CONTAINER} || true
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

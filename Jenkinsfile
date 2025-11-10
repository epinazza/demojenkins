pipeline {
    agent any

    environment {
        DOCKER_NETWORK = "jenkins-net"
        API_CONTAINER = "myapi-container"
        JMETER_CONTAINER = "jmeter-agent"
        RESULTS_DIR = "${WORKSPACE}/results"
        JMX_FILE = "API_TestPlan.jmx"
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
                sh "mkdir -p ${RESULTS_DIR}"
            }
        }

        stage('Build Docker Image') {
            steps {
                echo "🔧 Building Docker image..."
                sh "docker build -t myapi-img:v1 ."
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
                docker network create ${DOCKER_NETWORK} || true
                docker run -d --name ${API_CONTAINER} --network ${DOCKER_NETWORK} -p 8290:8290 -p 8253:8253 myapi-img:v1
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
                    if (status != "200") {
                        error "API not ready!"
                    }
                }
            }
        }

        stage('Check JMX File') {
            steps {
                echo "📄 Checking JMX file..."
                script {
                    if (!fileExists(JMX_FILE)) {
                        error "JMX file '${JMX_FILE}' not found!"
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
                docker run --rm -d --name ${JMETER_CONTAINER} \\
                    --network ${DOCKER_NETWORK} \\
                    -v ${WORKSPACE}:/tests \\
                    justb4/jmeter:latest \\
                    jmeter -n -t /tests/${JMX_FILE} -l /tests/results/report.jtl
                """
                sh "docker wait ${JMETER_CONTAINER}"
            }
        }

        stage('Archive JMeter Report') {
            steps {
                echo "📦 Archiving JMeter report..."
                archiveArtifacts artifacts: "results/*.jtl", allowEmptyArchive: false
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
    }
}

pipeline {
    agent any

    environment {
        DOCKER_IMAGE = "myapi-img:v1"
        CONTAINER_NAME = "myapi-container"
        NETWORK_NAME = "jenkins-net"
        API_URL = "http://host.docker.internal:8290/appointmentservices/getAppointment"
        JMETER_TEST_PLAN = "tests/API_TestPlan.jmx"
        JMETER_RESULTS_DIR = "results"
        THRESHOLD_MS = 500
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Prepare') {
            steps {
                echo "Workspace ready"
                sh "mkdir -p ${env.JMETER_RESULTS_DIR}"
            }
        }

        stage('Build Docker Image') {
            steps {
                echo "🔧 Building Docker image..."
                sh "docker build -t ${DOCKER_IMAGE} ."
            }
        }

        stage('Stop & Remove Old Container') {
            steps {
                echo "🧹 Cleaning up old container (if any)..."
                sh """
                    docker stop ${CONTAINER_NAME} || true
                    docker rm ${CONTAINER_NAME} || true
                """
            }
        }

        stage('Run New Container') {
            steps {
                echo "🚀 Starting WSO2 Micro Integrator container..."
                sh """
                    docker network create ${NETWORK_NAME} || true
                    docker run -d --name ${CONTAINER_NAME} --network ${NETWORK_NAME} -p 8290:8290 -p 8253:8253 ${DOCKER_IMAGE}
                """
            }
        }

        stage('Wait for API Ready') {
            steps {
                echo "⏳ Waiting for API to be ready..."
                retry(15) {
                    script {
                        def status = sh(script: "curl -s -o /dev/null -w %{http_code} ${API_URL}", returnStdout: true).trim()
                        echo "HTTP status: ${status}"
                        if (status != "200") {
                            echo "Waiting 2 seconds..."
                            sleep 2
                            error("API not ready yet")
                        }
                    }
                }
            }
        }

        stage('Check JMX File') {
            steps {
                echo "🔍 Checking if JMeter test plan exists..."
                script {
                    if (!fileExists(env.JMETER_TEST_PLAN)) {
                        error("JMeter test plan not found at ${env.JMETER_TEST_PLAN}")
                    }
                }
            }
        }

        stage('Load Test with JMeter') {
            steps {
                echo "⚙️ Running JMeter load test in Docker..."
                sh """
                    docker run --rm \
                        -v ${WORKSPACE}/tests:/tests \
                        -v ${WORKSPACE}/${JMETER_RESULTS_DIR}:/results \
                        justb4/jmeter:latest \
                        -n -t /tests/$(basename ${JMETER_TEST_PLAN}) \
                        -l /results/results.jtl -e -o /results/html | tee /results/summary.txt
                """
            }
        }

        stage('Evaluate Performance Threshold') {
            steps {
                echo "📊 Evaluating performance based on JMeter summary..."
                script {
                    def avgResponseTime = sh(
                        script: "grep -E 'summary =' ${JMETER_RESULTS_DIR}/summary.txt | awk '{print \$10}' | tail -n 1",
                        returnStdout: true
                    ).trim()

                    if (!avgResponseTime) {
                        error("Could not find average response time in summary report")
                    }

                    echo "Average response time: ${avgResponseTime} ms"

                    if (avgResponseTime.toFloat() > env.THRESHOLD_MS.toFloat()) {
                        error("Average response time ${avgResponseTime} ms exceeds threshold ${env.THRESHOLD_MS} ms")
                    }
                }
            }
        }
    }

    post {
        always {
            echo "🧹 Cleaning up Docker container..."
            sh """
                docker stop ${CONTAINER_NAME} || true
                docker rm ${CONTAINER_NAME} || true
            """
            echo "📦 Archiving JMeter report..."
            archiveArtifacts artifacts: "${JMETER_RESULTS_DIR}/**", allowEmptyArchive: true
        }
        failure {
            echo "❌ Pipeline failed!"
        }
        success {
            echo "✅ Pipeline succeeded!"
        }
    }
}

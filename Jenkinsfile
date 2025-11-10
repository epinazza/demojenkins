pipeline {
    agent any

    environment {
        JMETER_RESULTS_DIR = "results"
        JMETER_TEST_PLAN = "tests/API_TestPlan.jmx"
        CONTAINER_NAME = "myapi-container"
        IMAGE_NAME = "myapi-img:v1"
        API_URL = "http://host.docker.internal:8290/appointmentservices/getAppointment"
        MAX_RETRIES = 15
        SLEEP_SECONDS = 2
    }

    stages {
        stage('Checkout') {
            steps {
                echo "📥 Checking out repository..."
                checkout scm
            }
        }

        stage('Prepare') {
            steps {
                echo "🛠 Workspace ready"
                sh "mkdir -p ${JMETER_RESULTS_DIR}"
            }
        }

        stage('Build Docker Image') {
            steps {
                echo "🔧 Building Docker image..."
                sh """
                    docker build -t ${IMAGE_NAME} .
                """
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
                    docker network create jenkins-net || true
                    docker run -d --name ${CONTAINER_NAME} --network jenkins-net -p 8290:8290 -p 8253:8253 ${IMAGE_NAME}
                """
            }
        }

        stage('Wait for API Ready') {
            steps {
                echo "⏳ Waiting for API to be ready..."
                script {
                    def status = ""
                    retry(env.MAX_RETRIES.toInteger()) {
                        status = sh(
                            script: """curl -s -o /dev/null -w %{http_code} ${API_URL}""",
                            returnStdout: true
                        ).trim()
                        echo "HTTP status: ${status}"
                        if (status != "200") {
                            echo "Waiting ${SLEEP_SECONDS} seconds..."
                            sleep env.SLEEP_SECONDS.toInteger()
                            error("API not ready yet")
                        }
                    }
                }
            }
        }

        stage('Check JMX File') {
            steps {
                echo "🔍 Checking if JMeter test plan exists..."
                sh """
                    if [ ! -f ${JMETER_TEST_PLAN} ]; then
                        echo "❌ JMeter test plan not found!"
                        exit 1
                    fi
                """
            }
        }

        stage('Load Test with JMeter') {
            steps {
                echo "⚙️ Running JMeter load test in Docker..."
                sh """
                    docker run --rm \
                        -v \${env.WORKSPACE}/tests:/tests \
                        -v \${env.WORKSPACE}/${JMETER_RESULTS_DIR}:/results \
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
                    echo "Average response time: ${avgResponseTime} ms"

                    if (avgResponseTime != "" && avgResponseTime.toFloat() > 50) {
                        error("❌ Average response time exceeded 50 ms")
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

        success {
            echo "✅ Pipeline completed successfully!"
        }

        failure {
            echo "❌ Pipeline failed!"
        }
    }
}

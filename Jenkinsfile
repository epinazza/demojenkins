pipeline {
    agent any

    environment {
        JMETER_TEST_PLAN = "tests/API_TestPlan.jmx"
        JMETER_RESULTS_DIR = "results"
        API_CONTAINER_NAME = "myapi-container"
        API_IMAGE_NAME = "myapi-img:v1"
        API_PORT = 8290
    }

    stages {

        stage('Checkout SCM') {
            steps {
                checkout scm
            }
        }

        stage('Prepare') {
            steps {
                echo "Workspace ready: Jenkins will clone repository automatically"
            }
        }

        stage('Build Docker Image') {
            steps {
                echo "🔧 Building Docker image..."
                sh "docker build -t ${API_IMAGE_NAME} ."
            }
        }

        stage('Stop & Remove Old Container') {
            steps {
                echo "🧹 Cleaning up old container (if any)..."
                sh """
                    docker stop ${API_CONTAINER_NAME} || true
                    docker rm ${API_CONTAINER_NAME} || true
                """
            }
        }

        stage('Run New Container') {
            steps {
                echo "🚀 Starting WSO2 Micro Integrator container..."
                sh """
                    docker network create jenkins-net || true
                    docker run -d --name ${API_CONTAINER_NAME} --network jenkins-net -p ${API_PORT}:${API_PORT} -p 8253:8253 ${API_IMAGE_NAME}
                """
            }
        }

        stage('Wait for API Ready') {
            steps {
                echo "⏳ Waiting for API to be ready..."
                retry(10) {
                    script {
                        def status = sh(
                            script: "curl -s -o /dev/null -w '%{http_code}' http://localhost:${API_PORT}/appointmentservices/getAppointment",
                            returnStdout: true
                        ).trim()
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
                    if (!fileExists(JMETER_TEST_PLAN)) {
                        error("JMeter test plan not found at ${JMETER_TEST_PLAN}")
                    }
                }
            }
        }

        stage('Load Test with JMeter') {
            steps {
                echo "⚙️ Running JMeter load test in Docker..."
                sh """
                    mkdir -p ${JMETER_RESULTS_DIR}
                    docker run --rm \
                        -v '${env.WORKSPACE}/tests:/tests' \
                        -v '${env.WORKSPACE}/${JMETER_RESULTS_DIR}:/results' \
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
                    def summary = sh(
                        script: "grep -E 'summary =' ${JMETER_RESULTS_DIR}/summary.txt | awk '{print \$10}' | tail -n 1",
                        returnStdout: true
                    ).trim()

                    if (summary == "") {
                        echo "⚠️ Could not find average response time in summary report."
                    } else {
                        def avgResp = summary.toFloat()
                        echo "Average response time: ${avgResp} ms"
                        if (avgResp > 50) {
                            error("❌ Average response time exceeds threshold (50 ms)")
                        }
                    }
                }
            }
        }
    }

    post {
        always {
            echo "🧹 Cleaning up Docker container..."
            sh """
                docker stop ${API_CONTAINER_NAME} || true
                docker rm ${API_CONTAINER_NAME} || true
            """
            echo "📦 Archiving JMeter report..."
            archiveArtifacts artifacts: "${JMETER_RESULTS_DIR}/**", allowEmptyArchive: true
        }
        success {
            echo "✅ Pipeline succeeded!"
        }
        failure {
            echo "❌ Pipeline failed!"
        }
    }
}

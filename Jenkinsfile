pipeline {
    agent any

    environment {
        IMAGE_NAME = "myapi-img"
        CONTAINER_NAME = "myapi-container"
        NETWORK_NAME = "jenkins-net"
        API_PORT = "8290"
        MANAGEMENT_PORT = "8253"
        JMETER_TEST = "tests/API_TestPlan.jmx"
        JMETER_RESULT_JTL = "results/results.jtl"
        JMETER_RESULT_HTML = "results/html"
        JMETER_SUMMARY = "results/summary.txt"
        RESPONSE_THRESHOLD = "500" // milliseconds
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
                    curl -I http://localhost:8290 || true
                """
            }
        }

     stage('Load Test with JMeter') {
            steps {
                echo "⚙️ Running JMeter load test..."
                sh """
                    mkdir -p results
                    jmeter -n -t ${JMETER_TEST} -l ${JMETER_RESULT_JTL} -e -o ${JMETER_RESULT_HTML} | tee ${JMETER_SUMMARY}
                """
            }
        }

        stage('Evaluate Performance Threshold') {
            steps {
                echo "📊 Evaluating performance based on JMeter summary..."
                script {
                    def avgTime = sh(
                        script: "grep -E 'summary =' ${JMETER_SUMMARY} | awk '{print \$10}' | tail -n 1",
                        returnStdout: true
                    ).trim()

                    if (!avgTime) {
                        error("⚠️ Could not find average response time in summary report.")
                    }

                    echo "Average response time detected: ${avgTime} ms"

                    if (avgTime.toDouble() > RESPONSE_THRESHOLD.toDouble()) {
                        error("❌ Average response time (${avgTime} ms) exceeds ${RESPONSE_THRESHOLD} ms threshold!")
                    } else {
                        echo "✅ Performance PASSED: ${avgTime} ms ≤ ${RESPONSE_THRESHOLD} ms"
                    }
                }
            }
        }


    }

    post {
        always {
            echo "✅ Pipeline finished!"
        }
    }
}

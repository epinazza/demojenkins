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

        stage('Prepare') {
            steps {
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
                echo "⏳ Wait 30 seconds for WSO2 MI to fully start..."
                sh """
                    sleep 30
                    curl -I http://localhost:${API_PORT} || true
                """
            }
        }

        stage('Load Test with JMeter') {
            steps {
                echo "⚙️ Running JMeter load test in Docker..."
                sh """
                    mkdir -p results
                    docker run --rm \
                        -v $PWD/tests:/tests \
                        -v $PWD/results:/results \
                        justb4/jmeter:5.6.2 \
                        -n -t /tests/API_TestPlan.jmx \
                        -l /results/results.jtl \
                        -e -o /results/html | tee results/summary.txt
                """
            }
        }

        stage('Evaluate Performance Threshold') {
            steps {
                echo "📊 Evaluating performance based on JMeter summary..."
                script {
                    def avgResponse = sh(
                        script: "grep -E 'summary =' ${JMETER_SUMMARY} | awk '{print \$10}' | tail -n 1",
                        returnStdout: true
                    ).trim()
                    
                    if (!avgResponse) {
                        error("⚠️ Could not find average response time in summary report.")
                    } else {
                        echo "Average response time: ${avgResponse} ms"
                        if (avgResponse.toFloat() > RESPONSE_THRESHOLD.toFloat()) {
                            error("❌ Build failed: Average response time ${avgResponse} ms > ${RESPONSE_THRESHOLD} ms")
                        } else {
                            echo "✅ Performance within threshold."
                        }
                    }
                }
            }
        }
    }

    post {
        always {
            echo "🧹 Cleaning up..."
            sh """
                docker stop ${CONTAINER_NAME} || true
                docker rm ${CONTAINER_NAME} || true
            """
            echo "✅ Pipeline finished!"
        }
    }
}

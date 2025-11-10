pipeline {
    agent any

    environment {
        CONTAINER_NAME = "myapi-container"
        IMAGE_NAME = "myapi-img:v1"
        API_PORT = "8290"
        THRESHOLD_MS = 500
    }

    stages {

        stage('Prepare') {
            steps {
                echo "Workspace ready: Jenkins will clone repository automatically"
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
                    docker run -d --name ${CONTAINER_NAME} --network jenkins-net -p ${API_PORT}:${API_PORT} -p 8253:8253 ${IMAGE_NAME}
                """
            }
        }

        stage('Wait for API Ready') {
            steps {
                echo "⏳ Waiting for API to be ready..."
                retry(15) { // Retry 15 times with 2-second intervals
                    sh """
                        STATUS_CODE=\$(curl -s -o /dev/null -w %{http_code} http://localhost:${API_PORT}/appointmentservices/getAppointment || echo 000)
                        echo "HTTP status: \$STATUS_CODE"
                        if [ "\$STATUS_CODE" != "200" ]; then
                            echo "Waiting 2 seconds..."
                            sleep 2
                            exit 1
                        fi
                    """
                }
            }
        }

        stage('Load Test with JMeter') {
            steps {
                echo "⚙️ Running JMeter load test in Docker..."
                sh """
                    mkdir -p results
                    docker run --rm \
                        -v \$PWD/tests:/tests \
                        -v \$PWD/results:/results \
                        justb4/jmeter:latest \
                        -n -t /tests/API_TestPlan.jmx \
                        -l /results/results.jtl \
                        -e -o /results/html \
                    | tee results/summary.txt
                """
            }
        }

        stage('Evaluate Performance Threshold') {
            steps {
                echo "📊 Evaluating performance based on JMeter summary..."
                script {
                    def avgResponse = sh(
                        script: "grep -E 'summary =' results/summary.txt | awk '{print \$10}' | tail -n 1",
                        returnStdout: true
                    ).trim()
                    echo "Average response time: ${avgResponse} ms"

                    if (avgResponse != "" && avgResponse.toFloat() > THRESHOLD_MS.toFloat()) {
                        error "⚠️ Average response time ${avgResponse} ms exceeds threshold ${THRESHOLD_MS} ms!"
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
            echo "📦 Archiving JMeter report..."
            archiveArtifacts artifacts: 'results/html/**', allowEmptyArchive: true
        }

        failure {
            echo "❌ Pipeline failed!"
        }

        success {
            echo "✅ Pipeline finished successfully!"
        }
    }
}
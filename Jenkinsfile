pipeline {
    agent any

    environment {
        CONTAINER_NAME = "myapi-container"
        IMAGE_NAME = "myapi-img:v1"
        API_PORT = "8290"
        THRESHOLD_MS = 500
        WORKSPACE_DIR = "${WORKSPACE}"
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
                sh """
                    for i in {1..30}; do
                        STATUS_CODE=\$(docker exec ${CONTAINER_NAME} curl -s -o /dev/null -w "%{http_code}" http://localhost:${API_PORT}/appointmentservices/getAppointment || true)
                        if [ "\$STATUS_CODE" == "200" ]; then
                            echo "✅ API is ready!"
                            exit 0
                        fi
                        echo "Waiting... attempt \$i"
                        sleep 2
                    done
                    echo "❌ API did not start in time"
                    exit 1
                """
            }
        }

        stage('Check JMX File') {
            steps {
                echo "🔍 Checking JMeter test plan exists..."
                sh "ls -l ${WORKSPACE_DIR}/tests"
            }
        }

        stage('Load Test with JMeter') {
            steps {
                echo "⚙️ Running JMeter load test in Docker..."
                sh """
                    mkdir -p ${WORKSPACE_DIR}/results
                    docker run --rm \
                        -v ${WORKSPACE_DIR}/tests:/tests \
                        -v ${WORKSPACE_DIR}/results:/results \
                        justb4/jmeter:latest \
                        -n -t /tests/API_TestPlan.jmx \
                        -l /results/results.jtl \
                        -e -o /results/html
                """
            }
        }

        stage('Evaluate Performance Threshold') {
            steps {
                echo "📊 Evaluating performance based on JMeter summary..."
                script {
                    // Extract average response time from JMeter result
                    def avgResponse = sh(
                        script: "grep 'summary =' ${WORKSPACE_DIR}/results/results.jtl | awk '{print \$10}' | tail -n 1",
                        returnStdout: true
                    ).trim()

                    if (avgResponse == '') {
                        error "❌ Could not find average response time in JMeter results!"
                    }

                    echo "Average response time: ${avgResponse} ms"

                    if (avgResponse.toFloat() > THRESHOLD_MS.toFloat()) {
                        error "⚠️ Average response time ${avgResponse} ms exceeds threshold ${THRESHOLD_MS} ms!"
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

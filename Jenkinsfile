pipeline {
    agent any

    environment {
        DOCKER_IMAGE = "myapi-img:v1"
        CONTAINER_NAME = "myapi-container"
        NETWORK_NAME = "jenkins-net"
        JMETER_IMAGE = "justb4/jmeter:latest"
        TEST_PLAN_PATH = "/tests/API_TestPlan.jmx"
        RESULTS_DIR = "results"
        HTML_REPORT_DIR = "results/html"
        PORT_1 = "8290"
        PORT_2 = "8253"
    }

    stages {

        stage('Prepare') {
            steps {
                echo "Workspace ready: Jenkins will clone repository automatically"
            }
        }

        stage('Build Docker Image') {
            steps {
                echo "🔧 Building Docker image"
                sh "docker build -t ${DOCKER_IMAGE} ."
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
                    docker run -d --name ${CONTAINER_NAME} --network ${NETWORK_NAME} \
                    -p ${PORT_1}:${PORT_1} -p ${PORT_2}:${PORT_2} ${DOCKER_IMAGE}
                """
            }
        }

        stage('Test API') {
            steps {
                echo "⏳ Wait 30 seconds for WSO2 MI to fully start..."
                sh """
                    sleep 30
                    docker exec ${CONTAINER_NAME} curl -I http://localhost:${PORT_1} || true
                """
            }
        }

        stage('Load Test with JMeter') {
            steps {
                echo "⚙️ Running JMeter load test in Docker..."
                sh """
                    mkdir -p ${RESULTS_DIR}
                    docker run --rm -v ${WORKSPACE}/tests:/tests \
                    -v ${WORKSPACE}/${RESULTS_DIR}:/results \
                    ${JMETER_IMAGE} -n -t ${TEST_PLAN_PATH} \
                    -l /results/results.jtl -e -o /results/html | tee ${RESULTS_DIR}/summary.txt
                """
            }
        }

        stage('Evaluate Performance Threshold') {
            steps {
                echo "📊 Evaluating performance based on JMeter summary..."
                script {
                    def avgResponse = sh(
                        script: "grep -E 'summary =' ${RESULTS_DIR}/summary.txt | awk '{print \$10}' | tail -n 1",
                        returnStdout: true
                    ).trim()

                    if (!avgResponse) {
                        error "⚠️ Could not find average response time in summary report."
                    }

                    def threshold = 50
                    echo "Average Response Time: ${avgResponse} ms"

                    if (avgResponse.toFloat() > threshold) {
                        error "❌ Average response time (${avgResponse} ms) exceeded threshold (${threshold} ms)"
                    } else {
                        echo "✅ Performance within acceptable range (${avgResponse} ms ≤ ${threshold} ms)"
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
        }

        success {
            echo "📁 Archiving JMeter HTML report to Jenkins..."
            archiveArtifacts artifacts: "${HTML_REPORT_DIR}/**", allowEmptyArchive: true
            echo "✅ Pipeline finished successfully!"
        }

        failure {
            echo "⚠️ Pipeline failed! Archiving any existing results..."
            archiveArtifacts artifacts: "${HTML_REPORT_DIR}/**", allowEmptyArchive: true
        }
    }
}

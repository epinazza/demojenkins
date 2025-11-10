pipeline {
    agent any

    environment {
        DOCKER_NETWORK = "jenkins-net"
        JMX_FILE = "API_TestPlan.jmx"
        RESULTS_DIR = "${WORKSPACE}/results"
        TESTS_DIR = "${WORKSPACE}/tests"
        DOCKER_IMAGE = "myapi-img:v1"
        CONTAINER_NAME = "myapi-container"
    }

    stages {

        stage('Checkout SCM') {
            steps {
                echo "🔄 Checking out repository..."
                checkout scm
            }
        }

        stage('Prepare Workspace') {
            steps {
                echo "🛠 Preparing workspace..."
                sh """
                    mkdir -p ${RESULTS_DIR} ${TESTS_DIR}
                    cp ${WORKSPACE}/${JMX_FILE} ${TESTS_DIR}/
                """
            }
        }

        stage('Build Docker Image') {
            steps {
                echo "🔧 Building WSO2 Docker image..."
                sh "docker build -t ${DOCKER_IMAGE} ."
            }
        }

        stage('Stop & Remove Old API Container') {
            steps {
                echo "🧹 Cleaning up old WSO2 container..."
                sh """
                    docker stop ${CONTAINER_NAME} || true
                    docker rm ${CONTAINER_NAME} || true
                """
            }
        }

        stage('Run WSO2 Container') {
            steps {
                echo "🚀 Starting WSO2 Micro Integrator..."
                sh """
                    docker network create ${DOCKER_NETWORK} || true
                    docker run -d --name ${CONTAINER_NAME} \
                        --network ${DOCKER_NETWORK} \
                        -p 8290:8290 -p 8253:8253 \
                        ${DOCKER_IMAGE}
                """
            }
        }

        stage('Wait for API Ready') {
            steps {
                echo "⏳ Waiting 40 seconds for API..."
                sh "sleep 40"
                script {
                    def status = sh(
                        script: "curl -s -o /dev/null -w %{http_code} http://host.docker.internal:8290/appointmentservices/getAppointment",
                        returnStdout: true
                    ).trim()
                    echo "HTTP status: ${status}"
                    if (status != "200") {
                        error "API not ready!"
                    }
                }
            }
        }

        stage('Debug JMX File') {
            steps {
                echo "🔍 Debug: Listing tests folder..."
                sh "ls -l ${TESTS_DIR}"
            }
        }

        stage('Run JMeter Load Test') {
            steps {
                echo "🏃 Running JMeter load test..."
                sh """
                    mkdir -p ${RESULTS_DIR}
                    docker run --rm --name jmeter-agent \
                        --network ${DOCKER_NETWORK} \
                        -u root \
                        -v ${TESTS_DIR}:/tests \
                        -v ${RESULTS_DIR}:/tests/results \
                        -w /tests \
                        justb4/jmeter:latest \
                        -n -t ${JMX_FILE} -l results/report.jtl
                """
            }
        }

        stage('Archive JMeter Report') {
            steps {
                echo "📂 Archiving JMeter report..."
                archiveArtifacts artifacts: 'results/report.jtl', allowEmptyArchive: true
            }
        }

        stage('Evaluate Performance') {
            steps {
                echo "📊 Evaluating performance..."
                script {
                    def avgResp = sh(
                        script: "grep 'summary =' ${RESULTS_DIR}/report.jtl | awk '{print \$10}' | tail -n1",
                        returnStdout: true
                    ).trim()
                    echo "Average response time: ${avgResp} ms"
                    if (avgResp.toFloat() > 50) {
                        error "Average response time exceeded threshold! (${avgResp} ms > 50 ms)"
                    }
                }
            }
        }
    }

    post {
        always {
            echo "🧹 Cleaning up Docker containers..."
            sh """
                docker stop ${CONTAINER_NAME} || true
                docker rm ${CONTAINER_NAME} || true
            """
        }
        success {
            echo "✅ Pipeline succeeded!"
        }
        failure {
            echo "❌ Pipeline failed!"
        }
    }
}

pipeline {
    agent any

    environment {
        WORKSPACE_DIR = "${env.WORKSPACE}"
        TESTS_DIR = "${WORKSPACE_DIR}/tests"
        RESULTS_DIR = "${WORKSPACE_DIR}/results"
        DOCKER_NETWORK = "jenkins-net"
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
                    mkdir -p ${TESTS_DIR} ${RESULTS_DIR}
                    cp ${WORKSPACE_DIR}/API_TestPlan.jmx ${TESTS_DIR}/
                """
            }
        }

        stage('Build Docker Image') {
            steps {
                echo "🔧 Building WSO2 Docker image..."
                sh """
                    docker build -t ${DOCKER_IMAGE} .
                """
            }
        }

        stage('Stop & Remove Old Container') {
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
                    def httpStatus = sh(
                        script: "curl -s -o /dev/null -w %{http_code} http://host.docker.internal:8290/appointmentservices/getAppointment",
                        returnStdout: true
                    ).trim()
                    echo "HTTP status: ${httpStatus}"
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
                        -v ${WORKSPACE_DIR}:/tests \
                        -w /tests/tests \
                        justb4/jmeter:latest \
                        -n -t API_TestPlan.jmx \
                        -l ../results/report.jtl
                """
            }
        }

        stage('Archive JMeter Report') {
            steps {
                echo "📂 Archiving JMeter results..."
                archiveArtifacts artifacts: "results/**", allowEmptyArchive: true
            }
        }

        stage('Evaluate Performance') {
            steps {
                echo "📊 Evaluating performance..."
                script {
                    def avgResp = sh(
                        script: "grep -E 'summary =' ${RESULTS_DIR}/report.jtl | awk '{print \$10}' | tail -n 1",
                        returnStdout: true
                    ).trim()
                    echo "Average response time: ${avgResp} ms"
                    if (avgResp.toFloat() > 50) {
                        error "❌ Average response time exceeded 50ms!"
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
        failure {
            echo "❌ Pipeline failed!"
        }
        success {
            echo "✅ Pipeline completed successfully!"
        }
    }
}

pipeline {
    agent any

    environment {
        WORKSPACE_PATH = "/var/lib/docker/volumes/jenkins_home/_data/workspace/pipelineA"
        JMX_FILE = "${WORKSPACE_PATH}/API_TestPlan.jmx"
        RESULTS_DIR = "${WORKSPACE_PATH}/results"
        JMETER_IMAGE = "jmeter-with-plugins:latest"
        WSO2_IMAGE = "myapi-img:v1"
        WSO2_CONTAINER = "myapi-container"
        NETWORK_NAME = "jenkins-net"
    }

    stages {

        stage('Checkout SCM') {
            steps {
                checkout scm
            }
        }

        stage('Prepare Workspace') {
            steps {
                echo "🛠 Preparing workspace..."
                sh "mkdir -p ${RESULTS_DIR}"
            }
        }

        stage('Build WSO2 Docker Image') {
            steps {
                echo "🔧 Building WSO2 Docker image..."
                sh "docker build -t ${WSO2_IMAGE} ."
            }
        }

        stage('Stop & Remove Old WSO2 Container') {
            steps {
                echo "🧹 Cleaning up old WSO2 container (if any)..."
                sh """
                    docker stop ${WSO2_CONTAINER} || true
                    docker rm ${WSO2_CONTAINER} || true
                """
            }
        }

        stage('Run WSO2 Container') {
            steps {
                echo "🚀 Starting WSO2 Micro Integrator container..."
                sh """
                    docker network create ${NETWORK_NAME} || true
                    docker run -d --name ${WSO2_CONTAINER} --network ${NETWORK_NAME} -p 8290:8290 -p 8253:8253 ${WSO2_IMAGE}
                """
            }
        }

        stage('Wait for API Ready') {
            steps {
                echo "⏳ Waiting for API to be ready..."
                script {
                    retry(10) {
                        def status = sh(
                            script: "docker run --rm --network ${NETWORK_NAME} busybox sh -c 'wget -qO- http://${WSO2_CONTAINER}:8290/appointmentservices/getAppointment >/dev/null; echo \$?'",
                            returnStdout: true
                        ).trim()
                        if (status != "0") {
                            echo "Attempt API not ready yet, retrying..."
                            sleep 5
                            error("API not ready")
                        } else {
                            echo "✅ API is ready!"
                        }
                    }
                }
            }
        }

        stage('Prepare JMX for JMeter') {
            steps {
                echo "📄 Making JMX available for JMeter..."
                sh """
                    mkdir -p ${RESULTS_DIR}
                    cp ${JMX_FILE} ${WORKSPACE_PATH}/API_TestPlan.jmx
                    ls -l ${WORKSPACE_PATH}
                """
            }
        }

        stage('Run Load Test with JMeter') {
            steps {
                echo "🏃 Running JMeter load test..."
                sh """
                    docker run --rm --name jmeter-agent \
                        --network ${NETWORK_NAME} \
                        -v ${WORKSPACE_PATH}/API_TestPlan.jmx:/tests/API_TestPlan.jmx:ro \
                        -v ${RESULTS_DIR}:/tests/results:rw \
                        -w /tests ${JMETER_IMAGE} \
                        -n -t /tests/API_TestPlan.jmx \
                        -l /tests/results/report.jtl
                """
            }
        }

        stage('Archive JMeter Report') {
            steps {
                archiveArtifacts artifacts: 'results/**', allowEmptyArchive: true
            }
        }
    }

    post {
        always {
            echo "🧹 Cleaning up Docker containers..."
            sh """
                docker stop ${WSO2_CONTAINER} || true
                docker rm ${WSO2_CONTAINER} || true
                docker stop jmeter-agent || true
                docker rm jmeter-agent || true
            """
        }
    }
}

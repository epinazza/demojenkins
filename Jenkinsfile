pipeline {
    agent any

    environment {
        API_CONTAINER_NAME = "myapi-container"
        API_IMAGE = "myapi-img:v1"
        JMETER_IMAGE = "my-jmeter-img:latest"
        JMETER_CONTAINER_NAME = "jmeter-agent"
        NETWORK_NAME = "jenkins-net"
        WORKSPACE_DIR = "/var/lib/docker/volumes/jenkins_home/_data/workspace/${JOB_NAME}"
        RESULTS_DIR = "${WORKSPACE_DIR}/results"
        JMX_FILE = "${WORKSPACE_DIR}/API_TestPlan.jmx"
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
                sh "docker build -t ${API_IMAGE} -f Dockerfile ."
            }
        }

        stage('Build JMeter Docker Image') {
            steps {
                echo "⚡ Building custom JMeter image with BlazeMeter plugin..."
                sh "docker build -t ${JMETER_IMAGE} -f Dockerfile.jmeter ."
            }
        }

        stage('Stop & Remove Old WSO2 Container') {
            steps {
                echo "🧹 Cleaning up old WSO2 container (if any)..."
                sh "docker stop ${API_CONTAINER_NAME} || true"
                sh "docker rm ${API_CONTAINER_NAME} || true"
            }
        }

        stage('Run WSO2 Container') {
            steps {
                echo "🚀 Starting WSO2 Micro Integrator container..."
                sh "docker network create ${NETWORK_NAME} || true"
                sh "docker run -d --name ${API_CONTAINER_NAME} --network ${NETWORK_NAME} -p 8290:8290 -p 8253:8253 ${API_IMAGE}"
            }
        }

        stage('Wait for API Ready') {
            steps {
                echo "⏳ Waiting for API to be ready..."
                script {
                    retry(10) {
                        def status = sh(
                            script: "docker run --rm --network ${NETWORK_NAME} busybox sh -c 'wget -qO- http://${API_CONTAINER_NAME}:8290/appointmentservices/getAppointment >/dev/null; echo \$?'",
                            returnStdout: true
                        ).trim()
                        if (status != "0") {
                            echo "Attempt API not ready yet, retrying..."
                            sleep 5
                            error "API not ready"
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
                sh "cp ${JMX_FILE} ${WORKSPACE_DIR}/API_TestPlan.jmx"
            }
        }

        stage('Run Load Test with JMeter') {
            steps {
                echo "🏃 Running JMeter load test..."
                sh """
                docker run --rm --name ${JMETER_CONTAINER_NAME} \
                    --network ${NETWORK_NAME} \
                    -v ${JMX_FILE}:/tests/API_TestPlan.jmx:ro \
                    -v ${RESULTS_DIR}:/tests/results:rw \
                    -w /tests ${JMETER_IMAGE} \
                    -n -t /tests/API_TestPlan.jmx -l /tests/results/report.jtl
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
            sh "docker stop ${API_CONTAINER_NAME} || true"
            sh "docker rm ${API_CONTAINER_NAME} || true"
            sh "docker stop ${JMETER_CONTAINER_NAME} || true"
            sh "docker rm ${JMETER_CONTAINER_NAME} || true"
        }

        success {
            echo "✅ Pipeline completed successfully!"
        }

        failure {
            echo "❌ Pipeline failed!"
        }
    }
}

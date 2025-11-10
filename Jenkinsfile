pipeline {
    agent any

    environment {
        API_IMAGE = "myapi-img:v1"
        JMETER_IMAGE = "justb4/jmeter:latest"
        API_CONTAINER = "myapi-container"
        JMETER_CONTAINER = "jmeter-agent"
        NETWORK = "jenkins-net"
        JMX_FILE = "API_TestPlan.jmx"
        RESULTS_DIR = "results"
    }

    stages {
        stage('Clean Workspace') {
            steps {
                echo "🧹 Cleaning workspace..."
                deleteDir() // deletes everything in workspace
            }
        }
        stage('Checkout SCM') {
            steps {
                checkout([$class: 'GitSCM',
                    branches: [[name: '*/main']],
                    userRemoteConfigs: [[url: 'https://github.com/epinazza/demojenkins.git']]
                ])
            }
        }

        stage('Prepare Workspace') {
            steps {
                sh "mkdir -p ${RESULTS_DIR}"
            }
        }

        stage('Build API Docker Image') {
            steps {
                echo "🔧 Building WSO2 Docker image..."
                sh "docker build -t ${API_IMAGE} -f Dockerfile ."
            }
        }

        stage('Stop & Remove Old Containers') {
            steps {
                echo "🧹 Cleaning up old containers..."
                sh """
                    docker stop ${API_CONTAINER} || true
                    docker rm ${API_CONTAINER} || true
                    docker stop ${JMETER_CONTAINER} || true
                    docker rm ${JMETER_CONTAINER} || true
                """
            }
        }

        stage('Run API Container') {
            steps {
                echo "🚀 Starting WSO2 Micro Integrator container..."
                sh """
                    docker network create ${NETWORK} || true
                    docker run -d --name ${API_CONTAINER} --network ${NETWORK} -p 8290:8290 -p 8253:8253 ${API_IMAGE}
                """
            }
        }

        stage('Wait for API Ready') {
            steps {
                echo "⏳ Waiting for API to be ready..."
                retry(5) {
                    script {
                        def status = sh(script: "docker run --rm --network ${NETWORK} busybox sh -c 'wget -qO- http://${API_CONTAINER}:8290/appointmentservices/getAppointment >/dev/null; echo \$?'", returnStdout: true).trim()
                        if (status != "0") {
                            echo "Attempt API not ready yet, retrying..."
                            error("API not ready")
                        }
                    }
                }
                echo "✅ API is ready!"
            }
        }

        stage('Run JMeter Load Test') {
            steps {
                echo "🏃 Running JMeter load test..."
                sh """
                    docker run --rm --name ${JMETER_CONTAINER} \
                    --network ${NETWORK} \
                    -v \$(pwd)/${JMX_FILE}:/tests/${JMX_FILE}:ro \
                    -v \$(pwd)/${RESULTS_DIR}:/tests/results:rw \
                    -w /tests ${JMETER_IMAGE} \
                    -n -t /tests/${JMX_FILE} -l /tests/results/report.jtl
                """
            }
        }

        stage('Archive JMeter Report') {
            steps {
                archiveArtifacts artifacts: "${RESULTS_DIR}/report.jtl", allowEmptyArchive: true
            }
        }
    }

    post {
        always {
            echo "🧹 Cleaning up Docker containers..."
            sh """
                docker stop ${API_CONTAINER} || true
                docker rm ${API_CONTAINER} || true
                docker stop ${JMETER_CONTAINER} || true
                docker rm ${JMETER_CONTAINER} || true
            """
        }
        failure {
            echo "❌ Pipeline failed!"
        }
    }
}

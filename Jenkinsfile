pipeline {
    agent any

    environment {
        DOCKER_NETWORK = "jenkins-net"
        API_CONTAINER = "myapi-container"
        API_IMAGE = "myapi-img:v1"
        WORKSPACE_PATH = "${env.WORKSPACE}"
        RESULTS_DIR = "${WORKSPACE}/results"
        TESTS_DIR = "${WORKSPACE}/tests"
        JMX_FILE = "API_TestPlan.jmx"
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
                    cp ${WORKSPACE_PATH}/${JMX_FILE} ${TESTS_DIR}/
                """
            }
        }

        stage('Build Docker Image') {
            steps {
                echo "🔧 Building WSO2 Docker image..."
                sh "docker build -t ${API_IMAGE} ."
            }
        }

        stage('Stop & Remove Old API Container') {
            steps {
                echo "🧹 Cleaning up old WSO2 container..."
                sh """
                    docker stop ${API_CONTAINER} || true
                    docker rm ${API_CONTAINER} || true
                """
            }
        }

        stage('Run WSO2 Container') {
            steps {
                echo "🚀 Starting WSO2 Micro Integrator..."
                sh """
                    docker network create ${DOCKER_NETWORK} || true
                    docker run -d --name ${API_CONTAINER} --network ${DOCKER_NETWORK} -p 8290:8290 -p 8253:8253 ${API_IMAGE}
                """
            }
        }

        stage('Wait for API Ready') {
            steps {
                echo "⏳ Waiting 40 seconds for API..."
                sh "sleep 40"
                script {
                    def status = sh(script: "curl -s -o /dev/null -w %{http_code} http://host.docker.internal:8290/appointmentservices/getAppointment", returnStdout: true).trim()
                    echo "HTTP status: ${status}"
                    if (status != '200') {
                        error "API not ready, HTTP status ${status}"
                    }
                }
            }
        }

        stage('Debug JMX File') {
            steps {
                echo "🔍 Debug: Listing tests folder inside container..."
                sh """
                    ls -l ${TESTS_DIR}
                    ls -l ${TESTS_DIR}/${JMX_FILE}
                """
            }
        }

        stage('Run JMeter Load Test') {
            steps {
                echo "🏃 Running JMeter load test..."
                sh """
                    docker run --rm --name jmeter-agent \
                        --network ${DOCKER_NETWORK} \
                        -u root \
                        -v ${TESTS_DIR}:/tests \
                        -w /tests \
                        justb4/jmeter:latest \
                        -n -t ${JMX_FILE} -l results/report.jtl
                """
            }
        }

        stage('Archive JMeter Report') {
            steps {
                echo "📄 Archiving JMeter report..."
                archiveArtifacts artifacts: 'results/**', allowEmptyArchive: true
            }
        }

        stage('Evaluate Performance') {
            steps {
                echo "📊 Evaluating performance..."
                script {
                    def avgTime = sh(script: "grep -E 'summary =' ${RESULTS_DIR}/report.jtl | awk '{print \$10}' | tail -n 1", returnStdout: true).trim()
                    echo "Average Response Time: ${avgTime} ms"
                    if (avgTime.toFloat() > 50) {
                        error "❌ Average response time > 50ms, failing the build!"
                    }
                }
            }
        }
    }

    post {
        always {
            echo "🧹 Cleaning up Docker containers..."
            sh """
                docker stop ${API_CONTAINER} || true
                docker rm ${API_CONTAINER} || true
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

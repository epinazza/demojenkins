pipeline {
    agent any

    environment {
        DOCKER_NETWORK = "jenkins-net"
        RESULTS_DIR = "${WORKSPACE}/results"
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
                sh "docker build -t myapi-img:v1 ."
            }
        }

        stage('Stop & Remove Old WSO2 Container') {
            steps {
                echo "🧹 Cleaning up old WSO2 container (if any)..."
                sh """
                    docker stop myapi-container || true
                    docker rm myapi-container || true
                """
            }
        }

        stage('Run WSO2 Container') {
            steps {
                echo "🚀 Starting WSO2 Micro Integrator container..."
                sh """
                    docker network create ${DOCKER_NETWORK} || true
                    docker run -d --name myapi-container \
                        --network ${DOCKER_NETWORK} \
                        -p 8290:8290 -p 8253:8253 \
                        myapi-img:v1
                """
            }
        }

        stage('Wait for API Ready') {
            steps {
                echo "⏳ Waiting for API to be ready..."
                script {
                    def maxAttempts = 12
                    def attempt = 1
                    while (attempt <= maxAttempts) {
                        def status = sh(
                            script: "docker run --rm --network ${DOCKER_NETWORK} busybox sh -c 'wget -qO- http://myapi-container:8290/appointmentservices/getAppointment >/dev/null; echo \$?'",
                            returnStdout: true
                        ).trim()

                        if (status == "0") {
                            echo "✅ API is ready!"
                            break
                        } else {
                            echo "Attempt ${attempt}: API not ready yet"
                            sleep 5
                            attempt++
                        }

                        if (attempt > maxAttempts) {
                            error "API failed to start in time."
                        }
                    }
                }
            }
        }

        stage('Check JMX File') {
            steps {
                echo "📄 Checking JMX file..."
                sh "ls -l ${WORKSPACE}"
                script {
                    if (!fileExists("${WORKSPACE}/API_TestPlan.jmx")) {
                        error "JMX file not found!"
                    }
                }
            }
        }

        stage('Run Load Test with JMeter') {
            steps {
                echo "🏃 Running JMeter load test..."
                sh """
                    # Ensure workspace is accessible to container
                    chmod -R 777 ${WORKSPACE}
                    mkdir -p ${RESULTS_DIR}

                    docker run --rm --name jmeter-agent \
                        --network ${DOCKER_NETWORK} \
                        -v ${WORKSPACE}:/tests:rw \
                        -w /tests \
                        justb4/jmeter:latest \
                        -n -t /tests/API_TestPlan.jmx -l /tests/results/report.jtl
                """
            }
        }

        stage('Archive JMeter Report') {
            steps {
                archiveArtifacts artifacts: 'results/report.jtl', allowEmptyArchive: true
            }
        }
    }

    post {
        always {
            echo "🧹 Cleaning up Docker containers..."
            sh """
                docker stop myapi-container || true
                docker rm myapi-container || true
                docker stop jmeter-agent || true
                docker rm jmeter-agent || true
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

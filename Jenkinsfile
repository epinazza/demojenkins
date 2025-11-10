pipeline {
    agent any
    environment {
        RESULTS_DIR = "${WORKSPACE}/results"
        DOCKER_NETWORK = "jenkins-net"
        API_CONTAINER = "myapi-container"
        API_IMAGE = "myapi-img:v1"
        JMX_FILE = "API_TestPlan.jmx"
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
                sh "docker build -t ${API_IMAGE} ."
            }
        }

        stage('Stop & Remove Old WSO2 Container') {
            steps {
                echo "🧹 Cleaning up old WSO2 container (if any)..."
                sh """
                    docker stop ${API_CONTAINER} || true
                    docker rm ${API_CONTAINER} || true
                """
            }
        }

        stage('Run WSO2 Container') {
            steps {
                echo "🚀 Starting WSO2 Micro Integrator container..."
                sh """
                    docker network create ${DOCKER_NETWORK} || true
                    docker run -d --name ${API_CONTAINER} --network ${DOCKER_NETWORK} -p 8290:8290 -p 8253:8253 ${API_IMAGE}
                """
            }
        }

        stage('Wait for API Ready') {
            steps {
                echo "⏳ Waiting for API to be ready..."
                script {
                    int attempts = 0
                    int maxAttempts = 10
                    boolean apiReady = false

                    while(attempts < maxAttempts && !apiReady) {
                        def status = sh(script: "docker run --rm --network ${DOCKER_NETWORK} busybox sh -c 'wget -qO- http://${API_CONTAINER}:8290/appointmentservices/getAppointment >/dev/null; echo \$?'", returnStdout: true).trim()
                        if(status == "0") {
                            echo "API is ready!"
                            apiReady = true
                        } else {
                            attempts++
                            echo "Attempt ${attempts}: API not ready yet"
                            sleep 5
                        }
                    }

                    if(!apiReady) {
                        error "API did not become ready in time!"
                    }
                }
            }
        }

        stage('Check JMX File') {
            steps {
                echo "📄 Checking JMX file..."
                sh "ls -l ${WORKSPACE}"
                script {
                    if (!fileExists("${JMX_FILE}")) {
                        error "JMX file '${JMX_FILE}' not found in workspace!"
                    }
                }
            }
        }

        stage('Run Load Test with JMeter') {
            steps {
                echo "🏃 Running JMeter load test..."
                sh """
                    mkdir -p ${RESULTS_DIR}
                    docker run --rm --name jmeter-agent --network ${DOCKER_NETWORK} \
                        -v ${WORKSPACE}:/tests -w /tests \
                        justb4/jmeter:latest \
                        -n -t /tests/${JMX_FILE} -l /tests/results/report.jtl
                """
            }
        }

        stage('Archive JMeter Report') {
            steps {
                echo "📦 Archiving JMeter report..."
                archiveArtifacts artifacts: 'results/*.jtl', allowEmptyArchive: true
            }
        }
    }
    post {
        always {
            echo "🧹 Cleaning up Docker containers..."
            sh """
                docker stop ${API_CONTAINER} || true
                docker rm ${API_CONTAINER} || true
                docker stop jmeter-agent || true
                docker rm jmeter-agent || true
            """
        }
        success {
            echo "✅ Pipeline completed successfully!"
        }
        failure {
            echo "❌ Pipeline failed!"
        }
    }
}

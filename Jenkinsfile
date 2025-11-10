pipeline {
    agent any

    environment {
        WSO2_IMAGE = "myapi-img:v1"
        WSO2_CONTAINER = "myapi-container"
        JMETER_CONTAINER = "jmeter-agent"
        NETWORK_NAME = "jenkins-net"
        RESULTS_DIR = "${WORKSPACE}/results"
        JMX_FILE = "API_TestPlan.jmx" // Make sure this exists in your repo
        API_URL = "http://myapi-container:8290/appointmentservices/getAppointment"
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
                    docker run -d --name ${WSO2_CONTAINER} --network ${NETWORK_NAME} \
                        -p 8290:8290 -p 8253:8253 ${WSO2_IMAGE}
                """
            }
        }

        stage('Wait for API Ready') {
            steps {
                echo "⏳ Waiting for API to be ready..."
                script {
                    int retries = 20
                    int count = 0
                    def status = "0"
                    while (count < retries && status != "0") {
                        status = sh(script: "docker run --rm --network ${NETWORK_NAME} busybox sh -c 'wget -qO- ${API_URL} >/dev/null; echo \$?'", returnStdout: true).trim()
                        if (status == "0") {
                            echo "Attempt ${count + 1}: API is ready"
                            break
                        } else {
                            echo "Attempt ${count + 1}: API not ready yet"
                            sleep 5
                        }
                        count++
                    }
                    if (status != "0") {
                        error "API is not ready after ${retries * 5} seconds."
                    }
                }
            }
        }

        stage('Check JMX File') {
            steps {
                echo "📄 Checking JMX file..."
                script {
                    if (!fileExists("${JMX_FILE}")) {
                        error "JMX file ${JMX_FILE} not found!"
                    }
                }
            }
        }

        stage('Run Load Test with JMeter') {
            steps {
                echo "🏃 Running JMeter load test..."
                sh '''
                    # Ensure results folder exists
                    mkdir -p results

                    # Run JMeter inside Docker
                    docker run --rm --name jmeter-agent --network jenkins-net \
                        -v ${WORKSPACE}:/tests -w /tests \
                        justb4/jmeter:latest \
                        -n -t /tests/${JMX_FILE} -l /tests/results/report.jtl
                '''
            }
        }

        stage('Archive JMeter Report') {
            steps {
                echo "📦 Archiving JMeter report..."
                archiveArtifacts artifacts: 'results/*.jtl', allowEmptyArchive: false
            }
        }
    }

    post {
        always {
            echo "🧹 Cleaning up Docker containers..."
            sh """
                docker stop ${WSO2_CONTAINER} || true
                docker rm ${WSO2_CONTAINER} || true
                docker stop ${JMETER_CONTAINER} || true
                docker rm ${JMETER_CONTAINER} || true
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

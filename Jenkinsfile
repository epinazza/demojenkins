pipeline {
    agent any

    environment {
        DOCKER_NET = "jenkins-net"
        IMAGE_NAME = "myapi-img:v1"
        CONTAINER_NAME = "myapi-container"
        JMETER_CONTAINER = "jmeter-agent"
    }

    stages {

        stage('Declarative: Checkout SCM') {
            steps {
                checkout scm
            }
        }

        stage('Prepare Workspace') {
            steps {
                echo "🛠 Preparing workspace..."
                sh 'mkdir -p results'
            }
        }

        stage('Build WSO2 Docker Image') {
            steps {
                echo "🔧 Building WSO2 Docker image..."
                sh "docker build -t ${IMAGE_NAME} ."
            }
        }

        stage('Stop & Remove Old WSO2 Container') {
            steps {
                echo "🧹 Cleaning up old WSO2 container (if any)..."
                sh """
                    docker stop ${CONTAINER_NAME} || true
                    docker rm ${CONTAINER_NAME} || true
                """
            }
        }

        stage('Run WSO2 Container') {
            steps {
                echo "🚀 Starting WSO2 Micro Integrator container..."
                sh """
                    docker network create ${DOCKER_NET} || true
                    docker run -d --name ${CONTAINER_NAME} --network ${DOCKER_NET} -p 8290:8290 -p 8253:8253 ${IMAGE_NAME}
                """
            }
        }

        stage('Wait for API Ready') {
            steps {
                echo "⏳ Waiting for API to be ready..."
                script {
                    def ready = false
                    for (int i = 1; i <= 5; i++) {
                        def status = sh(script: "docker run --rm --network ${DOCKER_NET} busybox sh -c 'wget -qO- http://${CONTAINER_NAME}:8290/appointmentservices/getAppointment >/dev/null; echo \$?'", returnStdout: true).trim()
                        if (status == '0') {
                            echo "Attempt ${i}: API is ready (HTTP 200)"
                            ready = true
                            break
                        } else {
                            echo "Attempt ${i}: API not ready yet"
                            sleep 5
                        }
                    }
                    if (!ready) {
                        error "API did not become ready in time"
                    }
                }
            }
        }

        stage('Check JMX File') {
            steps {
                echo "📄 Checking JMX file..."
                script {
                    if (!fileExists('API_TestPlan.jmx')) {
                        error "JMX file not found in workspace!"
                    }
                }
            }
        }

        stage('Run Load Test with JMeter') {
            steps {
                echo '🏃 Running JMeter load test...'
                sh '''
                    # Ensure results folder exists
                    mkdir -p results

                    # Run JMeter Docker container
                    docker run --rm --name jmeter-agent \
                        --network jenkins-net \
                        -v ${WORKSPACE}:/tests -w /tests \
                        justb4/jmeter:latest \
                        -n -t /tests/API_TestPlan.jmx -l /tests/results/report.jtl
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
                docker stop ${CONTAINER_NAME} || true
                docker rm ${CONTAINER_NAME} || true
                docker stop ${JMETER_CONTAINER} || true
                docker rm ${JMETER_CONTAINER} || true
            """
        }
        failure {
            echo "❌ Pipeline failed!"
        }
    }
}

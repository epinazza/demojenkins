pipeline {
    agent any

    environment {
        DOCKER_NETWORK = 'jenkins-net'
        API_CONTAINER_NAME = 'myapi-container'
        API_IMAGE = 'myapi-img:v1'
        JMETER_CONTAINER_NAME = 'jmeter-agent'
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
                sh 'mkdir -p ${WORKSPACE}/results'
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
                    docker stop ${API_CONTAINER_NAME} || true
                    docker rm ${API_CONTAINER_NAME} || true
                """
            }
        }

        stage('Run WSO2 Container') {
            steps {
                echo "🚀 Starting WSO2 Micro Integrator container..."
                sh """
                    docker network create ${DOCKER_NETWORK} || true
                    docker run -d --name ${API_CONTAINER_NAME} --network ${DOCKER_NETWORK} \
                        -p 8290:8290 -p 8253:8253 ${API_IMAGE}
                """
            }
        }

        stage('Wait for API Ready') {
            steps {
                echo "⏳ Waiting for API to be ready..."
                script {
                    retry(10) {
                        def code = sh(
                            script: "docker run --rm --network ${DOCKER_NETWORK} busybox sh -c 'wget -qO- http://${API_CONTAINER_NAME}:8290/appointmentservices/getAppointment >/dev/null; echo \$?'",
                            returnStdout: true
                        ).trim()
                        if (code != '0') {
                            echo "Attempt API not ready yet, retrying..."
                            sleep 5
                            error("API not ready yet")
                        } else {
                            echo "✅ API is ready!"
                        }
                    }
                }
            }
        }

        stage('Run Load Test with JMeter') {
            steps {
                echo "🏃 Running JMeter load test..."
                sh """
                    chmod -R 777 ${WORKSPACE}
                    mkdir -p ${WORKSPACE}/results
                    docker run --rm --name ${JMETER_CONTAINER_NAME} --network ${DOCKER_NETWORK} \
                        -v ${WORKSPACE}/API_TestPlan.jmx:/tests/API_TestPlan.jmx:ro \
                        -v ${WORKSPACE}/results:/tests/results:rw \
                        -w /tests \
                        justb4/jmeter:latest \
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
            sh """
                docker stop ${API_CONTAINER_NAME} || true
                docker rm ${API_CONTAINER_NAME} || true
                docker stop ${JMETER_CONTAINER_NAME} || true
                docker rm ${JMETER_CONTAINER_NAME} || true
            """
        }

        success {
            echo "✅ Pipeline finished successfully!"
        }

        failure {
            echo "❌ Pipeline failed!"
        }
    }
}

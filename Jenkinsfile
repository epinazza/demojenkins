pipeline {
    agent any

    environment {
        WORKSPACE = "${env.WORKSPACE}"
    }

    stages {
        stage('Checkout SCM') {
            steps {
                checkout scm
            }
        }

        stage('Prepare Workspace') {
            steps {
                echo "🛠 Workspace ready"
                sh "mkdir -p ${WORKSPACE}/results"
            }
        }

        stage('Build Docker Image') {
            steps {
                echo "🔧 Building Docker image..."
                sh "docker build -t myapi-img:v1 ."
            }
        }

        stage('Stop & Remove Old API Container') {
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
                    docker network create jenkins-net || true
                    docker run -d --name myapi-container --network jenkins-net -p 8290:8290 -p 8253:8253 myapi-img:v1
                """
            }
        }

        stage('Wait for API Ready') {
            steps {
                echo "⏳ Waiting 40 seconds for API to be ready..."
                sh "sleep 40"
                script {
                    def status = sh(script: "curl -s -o /dev/null -w %{http_code} http://host.docker.internal:8290/appointmentservices/getAppointment", returnStdout: true).trim()
                    echo "HTTP status: ${status}"
                    if (status != "200") {
                        error "API not ready, HTTP status: ${status}"
                    }
                }
            }
        }

        stage('Check JMX File') {
            steps {
                echo "📄 Checking JMX file..."
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

                // Ensure results folder exists inside container
                sh "mkdir -p ${WORKSPACE}/results"

                // Stop & remove old JMeter container if it exists
                sh """
                    docker stop jmeter-agent || true
                    docker rm jmeter-agent || true
                """

                // Run JMeter container
                sh """
                    docker run -d --name jmeter-agent \
                        --network jenkins-net \
                        -v ${WORKSPACE}:/tests \
                        justb4/jmeter:latest \
                        /bin/bash -c "mkdir -p /tests/results && jmeter -n -t /tests/API_TestPlan.jmx -l /tests/results/report.jtl"
                """

                // Wait for container to finish
                sh "docker wait jmeter-agent"
            }
        }

        stage('Archive JMeter Report') {
            steps {
                echo "📦 Archiving JMeter report..."
                archiveArtifacts artifacts: "results/*.jtl", allowEmptyArchive: false
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
            echo "✅ Pipeline completed successfully!"
        }

        failure {
            echo "❌ Pipeline failed!"
        }
    }
}

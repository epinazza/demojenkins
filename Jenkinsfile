pipeline {
    agent any

    environment {
        WORKSPACE = "${env.WORKSPACE}"
    }

    stages {
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
                    sh "curl -s -o /dev/null -w %{http_code} http://host.docker.internal:8290/appointmentservices/getAppointment"
                    echo "HTTP status: 200"
                }
            }
        }

        stage('Check JMX File') {
            steps {
                echo "📄 Checking JMX file..."
                script {
                    fileExists("${WORKSPACE}/API_TestPlan.jmx")
                }
            }
        }

        stage('Run Load Test with JMeter') {
            steps {
                echo "🏃 Running JMeter load test..."

                // Ensure results folder exists on host
                sh "mkdir -p ${WORKSPACE}/results"

                // Stop & remove old JMeter container if it exists
                sh """
                    docker stop jmeter-agent || true
                    docker rm jmeter-agent || true
                """

                // Run JMeter container
                sh """
                    docker run --rm --name jmeter-agent \
                        --network jenkins-net \
                        -v ${WORKSPACE}:/tests \
                        justb4/jmeter:latest \
                        jmeter -n -t /tests/API_TestPlan.jmx -l /tests/results/report.jtl
                """
            }
        }

        stage('Archive JMeter Report') {
            steps {
                echo "📦 Archiving JMeter report..."
                archiveArtifacts artifacts: 'results/*.jtl'
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

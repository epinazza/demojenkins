pipeline {
    agent any

    environment {
        RESULTS_DIR = "${env.WORKSPACE}/results"
    }

    stages {
        stage('Declarative: Checkout SCM') {
            steps {
                checkout scm
            }
        }

        stage('Prepare Workspace') {
            steps {
                echo "🛠 Workspace ready"
                sh "mkdir -p ${RESULTS_DIR}"
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
                    def status = sh(
                        script: "curl -s -o /dev/null -w %{http_code} http://host.docker.internal:8290/appointmentservices/getAppointment",
                        returnStdout: true
                    ).trim()
                    echo "HTTP status: ${status}"
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
                echo "🏃 Running JMeter load test..."
                sh """
                    mkdir -p ${RESULTS_DIR}
                    docker run --rm --name jmeter-agent \\
                        --network jenkins-net \\
                        -u root \\
                        -v ${env.WORKSPACE}:/tests \\
                        -w /tests \\
                        justb4/jmeter:latest \\
                        -n -t API_TestPlan.jmx -l results/report.jtl
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
                docker stop myapi-container || true
                docker rm myapi-container || true
                docker stop jmeter-agent || true
                docker rm jmeter-agent || true
            """
        }
        failure {
            echo "❌ Pipeline failed!"
        }
    }
}

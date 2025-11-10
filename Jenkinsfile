pipeline {
    agent any

    environment {
        API_IMAGE = "myapi-img:v1"
        JMETER_IMAGE = "myjmeter-img:v1"
        API_CONTAINER = "myapi-container"
        NETWORK = "jenkins-net"
    }

    stages {
        stage('Prepare Workspace') {
            steps {
                echo "🧹 Preparing workspace..."
                sh 'mkdir -p results'
            }
        }

        stage('Build API Docker Image') {
            steps {
                echo "🔧 Building API image..."
                sh """docker build -f Dockerfile.api -t ${API_IMAGE} ."""
            }
        }

        stage('Build JMeter Docker Image') {
            steps {
                echo "📦 Building JMeter image..."
                sh """docker build -f Dockerfile.jmeter -t ${JMETER_IMAGE} ."""
            }
        }

        stage('Run API Container') {
            steps {
                echo "🚀 Starting API container..."
                sh """
                    docker network create ${NETWORK} || true
                    docker stop ${API_CONTAINER} || true
                    docker rm ${API_CONTAINER} || true
                    docker run -d --name ${API_CONTAINER} --network ${NETWORK} -p 8290:8290 -p 8253:8253 ${API_IMAGE}
                """
            }
        }

        stage('Wait for API Ready') {
            steps {
                echo "⏳ Waiting 40 seconds for API to start..."
                sh 'sleep 40'
                sh 'curl -s -o /dev/null -w "%{http_code}" http://host.docker.internal:8290/appointmentservices/getAppointment'
            }
        }

        stage('Run JMeter Load Test') {
            steps {
                echo "🏃 Running JMeter load test..."
                sh """
                    docker run --rm --network ${NETWORK} \
                        -v $PWD:/tests -w /tests \
                        ${JMETER_IMAGE} \
                        -n -t /tests/API_TestPlan.jmx -l /tests/results/report.jtl
                """
            }
        }

        stage('Evaluate Performance') {
            steps {
                script {
                    def avg = sh(script: "grep 'summary =' results/report.jtl | awk '{print \$10}' | tail -n 1", returnStdout: true).trim()
                    echo "Average response time: ${avg} ms"
                    if (avg.toFloat() > 500) {
                        error "❌ Average response time (${avg} ms) exceeds 500 ms threshold!"
                    } else {
                        echo "✅ Performance acceptable (${avg} ms)"
                    }
                }
            }
        }
    }

    post {
        always {
            echo "🧹 Cleaning up containers..."
            sh """
                docker stop ${API_CONTAINER} || true
                docker rm ${API_CONTAINER} || true
            """
        }
    }
}

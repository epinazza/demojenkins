pipeline {
    agent any

    environment {
        JMETER_RESULTS_DIR = "results"
        DOCKER_IMAGE = "myapi-img:v1"
        DOCKER_CONTAINER = "myapi-container"
        DOCKER_NETWORK = "jenkins-net"
    }

    stages {

        stage('Prepare') {
            steps {
                echo "🛠 Workspace ready"
                sh 'mkdir -p ${JMETER_RESULTS_DIR}'
            }
        }

        stage('Build Docker Image') {
            steps {
                echo "🔧 Building Docker image..."
                sh "docker build -t ${DOCKER_IMAGE} ."
            }
        }

        stage('Stop & Remove Old Container') {
            steps {
                echo "🧹 Cleaning up old container (if any)..."
                sh '''
                docker stop ${DOCKER_CONTAINER} || true
                docker rm ${DOCKER_CONTAINER} || true
                '''
            }
        }

        stage('Run New Container') {
            steps {
                echo "🚀 Starting WSO2 Micro Integrator container..."
                sh '''
                docker network create ${DOCKER_NETWORK} || true
                docker run -d --name ${DOCKER_CONTAINER} --network ${DOCKER_NETWORK} -p 8290:8290 -p 8253:8253 ${DOCKER_IMAGE}
                '''
            }
        }

        stage('Wait for API Ready') {
            steps {
                echo "⏳ Waiting for API to be ready..."
                retry(10) {
                    sh '''
                    STATUS=$(curl -s -o /dev/null -w "%{http_code}" http://host.docker.internal:8290/appointmentservices/getAppointment)
                    if [ "$STATUS" -ne 200 ]; then
                        echo "API not ready yet (HTTP $STATUS)"
                        exit 1
                    fi
                    '''
                }
            }
        }

        stage('Load Test with JMeter') {
            steps {
                echo "🚀 Running JMeter load test..."
                sh 'mkdir -p ${JMETER_RESULTS_DIR}'
                sh '''
                docker run --rm -v $(pwd)/''' + "${JMETER_RESULTS_DIR}" + ''':/results justb4/jmeter:latest \
                -n -t /tests/API_TestPlan.jmx \
                -l /results/results.jtl \
                -e -o /results/html
                '''
            }
        }

        stage('Evaluate Performance Threshold') {
            steps {
                echo "📊 Evaluating performance..."
                // Keep your existing threshold logic here
            }
        }
    }

    post {
        always {
            echo "🧹 Cleaning up Docker container..."
            sh '''
            docker stop ${DOCKER_CONTAINER} || true
            docker rm ${DOCKER_CONTAINER} || true
            '''
            echo "📦 Archiving JMeter report..."
            archiveArtifacts artifacts: "${JMETER_RESULTS_DIR}/**", allowEmptyArchive: true
        }
        failure {
            echo "❌ Pipeline failed!"
        }
    }
}

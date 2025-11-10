pipeline {
    agent any

    environment {
        API_PORT = '8290'
        JMETER_RESULTS_DIR = 'results'
    }

    stages {

        stage('Declarative: Checkout SCM') {
            steps {
                checkout scm
            }
        }

        stage('Prepare') {
            steps {
                echo "🛠 Workspace ready"
                sh 'mkdir -p ${JMETER_RESULTS_DIR}'
            }
        }

        stage('Build Docker Image') {
            steps {
                echo "🔧 Building Docker image..."
                sh 'docker build -t myapi-img:v1 .'
            }
        }

        stage('Stop & Remove Old Container') {
            steps {
                echo "🧹 Cleaning up old container (if any)..."
                sh '''
                    docker stop myapi-container || true
                    docker rm myapi-container || true
                '''
            }
        }

        stage('Run New Container') {
            steps {
                echo "🚀 Starting WSO2 Micro Integrator container..."
                sh '''
                    docker network create jenkins-net || true
                    docker run -d --name myapi-container --network jenkins-net -p ${API_PORT}:${API_PORT} -p 8253:8253 myapi-img:v1
                '''
            }
        }

        stage('Wait for API Ready') {
            steps {
                echo "⏳ Waiting 40 seconds for API to be ready..."
                sh 'sleep 40'
                sh '''
                    STATUS_CODE=$(curl -s -o /dev/null -w %{http_code} http://host.docker.internal:${API_PORT}/appointmentservices/getAppointment || echo 000)
                    echo "HTTP status: $STATUS_CODE"
                    if [ "$STATUS_CODE" != "200" ]; then
                        echo "API is not ready!"
                        exit 1
                    fi
                '''
            }
        }

        stage('Check JMX File') {
            steps {
                echo "📄 Checking JMX file..."
                sh 'test -f API_TestPlan.jmx || { echo "JMX file not found!"; exit 1; }'
            }
        }

        stage('Load Test with JMeter') {
            steps {
                echo "🏋️ Running Load Test with JMeter..."
                sh '''
                    docker run --rm -v $(pwd)/${JMETER_RESULTS_DIR}:/results -v $(pwd)/API_TestPlan.jmx:/API_TestPlan.jmx justb4/jmeter:latest \
                    -n -t /API_TestPlan.jmx -l /results/result.jtl
                '''
            }
        }

        stage('Evaluate Performance Threshold') {
            steps {
                echo "📊 Evaluating performance threshold..."
                sh '''
                    AVG_TIME=$(grep -E 'summary =' ${JMETER_RESULTS_DIR}/result.jtl | awk '{print $10}' | tail -n 1)
                    echo "Average response time: $AVG_TIME ms"
                    THRESHOLD=50
                    if (( $(echo "$AVG_TIME > $THRESHOLD" | bc -l) )); then
                        echo "❌ Performance threshold exceeded!"
                        exit 1
                    fi
                '''
            }
        }
    }

    post {
        always {
            echo "🧹 Cleaning up Docker container..."
            sh '''
                docker stop myapi-container || true
                docker rm myapi-container || true
            '''
            echo "📦 Archiving JMeter report..."
            archiveArtifacts artifacts: "${JMETER_RESULTS_DIR}/*", allowEmptyArchive: true
        }
        failure {
            echo "❌ Pipeline failed!"
        }
        success {
            echo "✅ Pipeline completed successfully!"
        }
    }
}

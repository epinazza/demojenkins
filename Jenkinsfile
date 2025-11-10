pipeline {
    agent any
    environment {
        API_PORT = "8290"
        JMETER_RESULTS_DIR = "results"
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
                echo "⏳ Waiting for API to be ready..."
                retry(15) {
                    sh '''
                        STATUS_CODE=\\$(curl -s -o /dev/null -w %{http_code} http://host.docker.internal:${API_PORT}/appointmentservices/getAppointment || echo 000)
                        echo "HTTP status: \\$STATUS_CODE"
                        if [ "\\$STATUS_CODE" != "200" ]; then
                            echo "Waiting 2 seconds..."
                            sleep 2
                            exit 1
                        fi
                    '''
                }
            }
        }

        stage('Check JMX File') {
            steps {
                echo "🔍 Checking if JMeter JMX file exists..."
                sh '''
                    if [ ! -f test2.jmx ]; then
                        echo "JMX file not found!"
                        exit 1
                    fi
                '''
            }
        }

        stage('Load Test with JMeter') {
            steps {
                echo "🏋️ Running JMeter test..."
                sh '''
                    docker run --rm -v $(pwd)/${JMETER_RESULTS_DIR}:/results -v $(pwd)/test2.jmx:/test2.jmx jmeter_image \
                    jmeter -n -t /test2.jmx -l /results/results.jtl -e -o /results/report
                '''
            }
        }

        stage('Evaluate Performance Threshold') {
            steps {
                echo "📊 Evaluating performance..."
                sh '''
                    AVG_RESP=\\$(grep -E 'summary =' ${JMETER_RESULTS_DIR}/summary.txt | awk '{print $10}' | tail -n 1)
                    echo "Average response time: \\$AVG_RESP"
                    THRESHOLD=50
                    awk -v avg="\\$AVG_RESP" -v th="$THRESHOLD" 'BEGIN { if (avg>th) exit 1 }'
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
            archiveArtifacts artifacts: "${JMETER_RESULTS_DIR}/**", allowEmptyArchive: true
        }
        failure {
            echo "❌ Pipeline failed!"
        }
    }
}

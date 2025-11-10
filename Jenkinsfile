pipeline {
    agent any
    environment {
        JMETER_RESULTS_DIR = 'results'
        TEST_PLAN = 'tests/API_TestPlan.jmx'
    }
    stages {

        stage('Checkout') {
            steps {
                echo "📥 Checking out repository..."
                checkout scm
            }
        }

        stage('Prepare') {
            steps {
                echo "🛠 Workspace ready"
                sh "mkdir -p ${JMETER_RESULTS_DIR}"
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
                    docker run -d --name myapi-container --network jenkins-net -p 8290:8290 -p 8253:8253 myapi-img:v1
                '''
            }
        }

        stage('Wait for API Ready') {
            steps {
                echo "⏳ Waiting for API to be ready..."
                script {
                    retry(15) {
                        sh '''
                            STATUS_CODE=$(curl -s -o /dev/null -w %{http_code} http://myapi-container:8290/appointmentservices/getAppointment)
                            echo "HTTP status: $STATUS_CODE"
                            if [ "$STATUS_CODE" != "200" ]; then
                                echo "Waiting 2 seconds..."
                                sleep 2
                                exit 1
                            fi
                        '''
                    }
                }
            }
        }

        stage('Check JMX File') {
            steps {
                echo "🔍 Checking if JMeter test plan exists..."
                sh """
                    if [ ! -f ${TEST_PLAN} ]; then
                        echo "JMeter test plan not found!"
                        exit 1
                    fi
                """
            }
        }

        stage('Load Test with JMeter') {
            steps {
                echo "⚙️ Running JMeter load test in Docker..."
                sh """
                    docker run --rm -v \$(pwd)/tests:/tests -v \$(pwd)/${JMETER_RESULTS_DIR}:/results justb4/jmeter:latest \\
                        -n -t /tests/$(basename ${TEST_PLAN}) -l /results/results.jtl -e -o /results/html | tee ${JMETER_RESULTS_DIR}/summary.txt
                """
            }
        }

        stage('Evaluate Performance Threshold') {
            steps {
                echo "📊 Evaluating performance based on JMeter summary..."
                script {
                    def avgResp = sh(script: "grep -E 'summary =' ${JMETER_RESULTS_DIR}/summary.txt | awk '{print \$10}' | tail -n 1", returnStdout: true).trim()
                    echo "Average response time: ${avgResp} ms"
                    if (avgResp.toFloat() > 50) {
                        error("❌ Average response time exceeded threshold (50ms)")
                    }
                }
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

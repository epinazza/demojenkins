pipeline {
    agent any

    environment {
        JMETER_TEST_PLAN = 'tests/API_TestPlan.jmx'
        JMETER_RESULTS_DIR = 'results'
    }

    stages {
        stage('Checkout SCM') {
            steps {
                checkout scm
            }
        }

        stage('Prepare') {
            steps {
                echo "Workspace ready: Jenkins will clone repository automatically"
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
                retry(15) {
                    script {
                        def status = sh(
                            script: 'curl -s -o /dev/null -w "%{http_code}" http://localhost:8290/appointmentservices/getAppointment',
                            returnStdout: true
                        ).trim()

                        if (status != "200") {
                            echo "HTTP status: ${status}, waiting 2 seconds..."
                            sleep 2
                            error("API not ready yet")
                        } else {
                            echo "HTTP status: ${status} ✅ API is ready"
                        }
                    }
                }
            }
        }

        stage('Check JMX File') {
            steps {
                echo "🔍 Checking if JMeter test plan exists..."
                sh '''
                    if [ ! -f ${JMETER_TEST_PLAN} ]; then
                        echo "JMeter test plan not found!"
                        exit 1
                    fi
                '''
            }
        }

        stage('Load Test with JMeter') {
            steps {
                echo "⚙️ Running JMeter load test in Docker..."
                sh """
                    mkdir -p ${JMETER_RESULTS_DIR}
                    docker run --rm \
                        -v "${WORKSPACE}/tests:/tests" \
                        -v "${WORKSPACE}/${JMETER_RESULTS_DIR}:/results" \
                        justb4/jmeter:latest \
                        -n -t /tests/$(basename ${JMETER_TEST_PLAN}) \
                        -l /results/results.jtl -e -o /results/html | tee /results/summary.txt
                """
            }
        }

        stage('Evaluate Performance Threshold') {
            steps {
                echo "📊 Evaluating performance based on JMeter summary..."
                script {
                    def avgResponseTime = sh(
                        script: "grep -E 'summary =' ${JMETER_RESULTS_DIR}/summary.txt | awk '{print \\$10}' | tail -n 1",
                        returnStdout: true
                    ).trim()

                    echo "Average response time: ${avgResponseTime} ms"

                    if (avgResponseTime != "" && avgResponseTime.toFloat() > 500) {
                        error("❌ Average response time ${avgResponseTime} ms exceeded threshold of 500 ms")
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

            echo "✅ Pipeline finished!"
        }

        failure {
            echo "❌ Pipeline failed!"
        }
    }
}

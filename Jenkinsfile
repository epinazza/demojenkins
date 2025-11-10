pipeline {
    agent any

    environment {
        TESTS_DIR = "${WORKSPACE}/tests"
        RESULTS_DIR = "${WORKSPACE}/results"
        JMETER_IMAGE = "justb4/jmeter:latest"
        WSO2_IMAGE = "myapi-img:v1"
        CONTAINER_NAME = "myapi-container"
        NETWORK_NAME = "jenkins-net"
        API_URL = "http://host.docker.internal:8290/appointmentservices/getAppointment"
        MAX_AVG_RESPONSE_TIME = 500
    }

    stages {
        stage('Checkout SCM') {
            steps {
                echo "🔄 Checking out repository..."
                checkout scm
            }
        }

        stage('Prepare Workspace') {
            steps {
                echo "🛠 Preparing workspace..."
                sh """
                    mkdir -p ${TESTS_DIR} ${RESULTS_DIR}
                    cp ${WORKSPACE}/API_TestPlan.jmx ${TESTS_DIR}/
                    chmod -R 777 ${TESTS_DIR} ${RESULTS_DIR}
                """
            }
        }

        stage('Build WSO2 Docker Image') {
            steps {
                echo "🔧 Building WSO2 Docker image..."
                sh "docker build -t ${WSO2_IMAGE} ."
            }
        }

        stage('Stop & Remove Old Container') {
            steps {
                echo "🧹 Cleaning up old WSO2 container..."
                sh """
                    docker stop ${CONTAINER_NAME} || true
                    docker rm ${CONTAINER_NAME} || true
                """
            }
        }

        stage('Run WSO2 Container') {
            steps {
                echo "🚀 Starting WSO2 Micro Integrator..."
                sh """
                    docker network create ${NETWORK_NAME} || true
                    docker run -d --name ${CONTAINER_NAME} --network ${NETWORK_NAME} \
                        -p 8290:8290 -p 8253:8253 ${WSO2_IMAGE}
                """
            }
        }

        stage('Wait for API Ready') {
            steps {
                echo "⏳ Waiting 40 seconds for API..."
                sh "sleep 40"
                script {
                    def status = sh(
                        script: "curl -s -o /dev/null -w %{http_code} ${API_URL}",
                        returnStdout: true
                    ).trim()
                    if (status != "200") {
                        error "API is not ready! HTTP status: ${status}"
                    }
                    echo "API is ready! HTTP status: ${status}"
                }
            }
        }

       stage('Run JMeter Load Test') {
            steps {
                echo "🏃 Running JMeter load test..."

                sh """
                    echo "🧩 Preparing test folders..."
                    mkdir -p ${WORKSPACE}/tests
                    mkdir -p ${WORKSPACE}/results
                    cp ${WORKSPACE}/API_TestPlan.jmx ${WORKSPACE}/tests/
                    chmod -R 777 ${WORKSPACE}/tests ${WORKSPACE}/results

                    echo "🔍 Checking that JMX file exists before container run..."
                    ls -l ${WORKSPACE}/tests

                    echo "🚀 Running JMeter test inside container..."
                    docker run --rm --name jmeter-agent \
                        --network jenkins-net \
                        -v ${WORKSPACE}/tests:/tests \
                        -v ${WORKSPACE}/results:/results \
                        justb4/jmeter:latest \
                        -n -t /tests/API_TestPlan.jmx -l /results/report.jtl
                """
            }
        }

        stage('Evaluate Performance') {
            steps {
                echo "📊 Evaluating performance..."
                script {
                    def avgRespTime = sh(
                        script: "awk -F',' '{sum+=\$2; count+=1} END {if(count>0) print sum/count; else print 0}' ${RESULTS_DIR}/report.jtl",
                        returnStdout: true
                    ).trim().toFloat()
                    echo "Average response time: ${avgRespTime} ms"

                    if (avgRespTime > MAX_AVG_RESPONSE_TIME) {
                        error "🚨 Average response time ${avgRespTime}ms exceeds threshold ${MAX_AVG_RESPONSE_TIME}ms!"
                    }
                }
            }
        }
    }

    post {
        always {
            echo "🧹 Cleaning up Docker containers..."
            sh """
                docker stop ${CONTAINER_NAME} || true
                docker rm ${CONTAINER_NAME} || true
            """
        }
        failure {
            echo "❌ Pipeline failed!"
        }
        success {
            echo "✅ Pipeline completed successfully!"
        }
    }
}

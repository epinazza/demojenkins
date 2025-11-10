pipeline {
    agent any

    environment {
        WORKSPACE_DIR = "${env.WORKSPACE}"
        JMETER_RESULTS = "${env.WORKSPACE}/results"
        TESTS_DIR = "${env.WORKSPACE}/tests"
        JMETER_IMAGE = "justb4/jmeter:latest"
        WSO2_IMAGE = "myapi-img:v1"
        WSO2_CONTAINER = "myapi-container"
        DOCKER_NETWORK = "jenkins-net"
    }

    stages {

        stage('Checkout SCM') {
            steps {
                checkout scm
            }
        }

        stage('Prepare Workspace') {
            steps {
                echo "🛠 Preparing workspace..."
                sh """
                    mkdir -p ${TESTS_DIR} ${JMETER_RESULTS}
                    cp ${WORKSPACE_DIR}/API_TestPlan.jmx ${TESTS_DIR}/
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
                    docker stop ${WSO2_CONTAINER} || true
                    docker rm ${WSO2_CONTAINER} || true
                """
            }
        }

        stage('Run WSO2 Container') {
            steps {
                echo "🚀 Starting WSO2 Micro Integrator..."
                sh """
                    docker network create ${DOCKER_NETWORK} || true
                    docker run -d --name ${WSO2_CONTAINER} --network ${DOCKER_NETWORK} -p 8290:8290 -p 8253:8253 ${WSO2_IMAGE}
                """
            }
        }

        stage('Wait for API Ready') {
            steps {
                echo "⏳ Waiting 40 seconds for API..."
                sh "sleep 40"

                script {
                    def status = sh(script: "curl -s -o /dev/null -w '%{http_code}' http://host.docker.internal:8290/appointmentservices/getAppointment", returnStdout: true).trim()
                    if (status != "200") {
                        error "API is not ready. HTTP status: ${status}"
                    }
                    echo "API is ready! HTTP status: ${status}"
                }
            }
        }

        stage('Run JMeter Load Test') {
            steps {
                echo "🏃 Running JMeter load test..."
                sh """
                    chmod -R 777 ${JMETER_RESULTS}
                    docker run --rm --name jmeter-agent \
                        --network ${DOCKER_NETWORK} \
                        -u root \
                        -v ${TESTS_DIR}:/tests \
                        -v ${JMETER_RESULTS}:/results \
                        -w /tests \
                        ${JMETER_IMAGE} \
                        -n -t API_TestPlan.jmx -l /results/report.jtl
                """
            }
        }

        stage('Evaluate Performance') {
            steps {
                script {
                    def avgResponse = sh(
                        script: "awk -F',' '{sum+=\$2; count++} END {if(count>0) print sum/count; else print 0}' ${JMETER_RESULTS}/report.jtl",
                        returnStdout: true
                    ).trim()

                    echo "Average Response Time: ${avgResponse} ms"

                    if (avgResponse.toFloat() > 500) {
                        error "❌ Average response time exceeds 500 ms!"
                    } else {
                        echo "✅ Performance is within limit."
                    }
                }
            }
        }
    }

    post {
        always {
            echo "🧹 Cleaning up Docker containers..."
            sh """
                docker stop ${WSO2_CONTAINER} || true
                docker rm ${WSO2_CONTAINER} || true
            """
        }
        failure {
            echo "❌ Pipeline failed!"
        }
    }
}

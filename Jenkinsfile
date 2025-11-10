pipeline {
    agent any
    environment {
        WORKSPACE_DIR = "${env.WORKSPACE}"
        DOCKER_NET = "jenkins-net"
        API_IMG = "myapi-img:v1"
        API_CONTAINER = "myapi-container"
        JMX_FILE = "API_TestPlan.jmx"
        RESULTS_DIR = "${WORKSPACE_DIR}/results"
        TESTS_DIR = "${WORKSPACE_DIR}/tests"
    }
    stages {
        stage('Checkout SCM') {
            steps {
                checkout scm
            }
        }

        stage('Prepare Workspace') {
            steps {
                echo "🛠 Workspace ready"
                sh """
                    mkdir -p ${RESULTS_DIR} ${TESTS_DIR}
                    cp ${WORKSPACE_DIR}/${JMX_FILE} ${TESTS_DIR}/
                """
            }
        }

        stage('Build Docker Image') {
            steps {
                echo "🔧 Building Docker image..."
                sh "docker build -t ${API_IMG} ."
            }
        }

        stage('Stop & Remove Old API Container') {
            steps {
                echo "🧹 Cleaning up old WSO2 container (if any)..."
                sh """
                    docker stop ${API_CONTAINER} || true
                    docker rm ${API_CONTAINER} || true
                """
            }
        }

        stage('Run WSO2 Container') {
            steps {
                echo "🚀 Starting WSO2 Micro Integrator container..."
                sh """
                    docker network create ${DOCKER_NET} || true
                    docker run -d --name ${API_CONTAINER} --network ${DOCKER_NET} \
                        -p 8290:8290 -p 8253:8253 ${API_IMG}
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

        stage('Debug JMX Inside Container') {
            steps {
                echo "🔍 Listing tests folder inside container..."
                sh "docker run --rm --name jmeter-debug --network ${DOCKER_NET} -u root -v ${TESTS_DIR}:/tests -w /tests busybox ls -l"
            }
        }

        stage('Run Load Test with JMeter') {
            steps {
                echo "🏃 Running JMeter load test..."
                sh """
                    docker run --rm --name jmeter-agent --network ${DOCKER_NET} -u root \
                        -v ${TESTS_DIR}:/tests -w /tests justb4/jmeter:latest \
                        -n -t ${JMX_FILE} -l results/report.jtl
                """
            }
        }

        stage('Archive JMeter Report') {
            steps {
                archiveArtifacts artifacts: 'results/report.jtl', allowEmptyArchive: true
            }
        }

        stage('Evaluate Performance') {
            steps {
                script {
                    // Extract average response time from JMeter JTL
                    def avgResponseTime = sh(
                        script: "awk -F',' 'NR>1{sum+=\$2; count++} END{if(count>0) print sum/count; else print 0}' ${RESULTS_DIR}/report.jtl",
                        returnStdout: true
                    ).trim()
                    echo "Average Response Time: ${avgResponseTime} ms"

                    // Fail build if avg > 500ms
                    if (avgResponseTime.toFloat() > 500) {
                        error "❌ Build failed: Average response time ${avgResponseTime} ms exceeds 500ms"
                    }
                }
            }
        }
    }

    post {
        always {
            echo "🧹 Cleaning up Docker containers..."
            sh """
                docker stop ${API_CONTAINER} || true
                docker rm ${API_CONTAINER} || true
                docker stop jmeter-agent || true
                docker rm jmeter-agent || true
            """
        }
    }
}

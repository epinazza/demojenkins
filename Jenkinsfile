pipeline {
    agent any

    environment {
        API_IMAGE = "myapi-img:v1"
        JMETER_IMAGE = "justb4/jmeter:5.5"
        API_CONTAINER = "myapi-container"
        JMETER_CONTAINER = "jmeter-agent"
        NETWORK = "jenkins-net"
        JMX_FILE = "API_TestPlan.jmx"
        RESULTS_DIR = "results"
    }

    stages {

        stage('Clean Workspace') {
            steps {
                echo "🧹 Cleaning workspace..."
                deleteDir()
            }
        }

        stage('Checkout SCM') {
            steps {
                checkout([$class: 'GitSCM',
                    branches: [[name: '*/main']],
                    userRemoteConfigs: [[url: 'https://github.com/epinazza/demojenkins.git']]
                ])
            }
        }

        stage('Prepare Workspace') {
            steps {
                echo "📂 Preparing workspace directories..."
                sh "mkdir -p ${RESULTS_DIR}"
                sh "rm -rf ${RESULTS_DIR}/* || true"
            }
        }

        stage('Build API Docker Image') {
            steps {
                echo "🔧 Building API Docker image..."
                sh "docker build -t ${API_IMAGE} -f Dockerfile ."
            }
        }

        stage('Stop & Remove Old Containers') {
            steps {
                echo "🧹 Cleaning up old containers..."
                sh """
                    docker stop ${API_CONTAINER} || true
                    docker rm ${API_CONTAINER} || true
                    docker stop ${JMETER_CONTAINER} || true
                    docker rm ${JMETER_CONTAINER} || true
                """
            }
        }

        stage('Run API Container') {
            steps {
                echo "🚀 Starting API container..."
                sh """
                    docker network create ${NETWORK} || true
                    docker run -d --name ${API_CONTAINER} --network ${NETWORK} -p 8290:8290 -p 8253:8253 ${API_IMAGE}
                """
            }
        }

        stage('Wait for API Ready') {
            steps {
                echo "⏳ Waiting for API to be ready..."
                script {
                    retry(10) {
                        def status = sh(
                            script: "docker run --rm --network ${NETWORK} busybox sh -c 'wget -qO- http://${API_CONTAINER}:8290/appointmentservices/getAppointment >/dev/null; echo \$?'",
                            returnStdout: true
                        ).trim()
                        if (status != "0") {
                            echo "Attempt API not ready yet, retrying..."
                            sleep 5
                            error "API not ready"
                        } else {
                            echo "✅ API is ready!"
                        }
                    }
                }
            }
        }

        stage('Verify JMX File in Workspace') {
            steps {
                echo "🔍 Checking for JMX file in workspace..."
                sh '''
                    echo "Workspace path: ${WORKSPACE}"
                    echo "Listing workspace content recursively:"
                    ls -R ${WORKSPACE} || true
                '''
            }
        }

        stage('Run JMeter Load Test') {
            steps {
                echo "🏃 Running JMeter load test..."
                sh '''
                    # Find the JMX file anywhere in workspace
                    JMX_PATH=$(find ${WORKSPACE} -type f -name "${JMX_FILE}" | head -n 1)

                    if [ -z "$JMX_PATH" ]; then
                        echo "❌ Could not find ${JMX_FILE}"
                        exit 1
                    fi

                    echo "✅ Found JMX file at: $JMX_PATH"
                    echo "🧪 Running JMeter test inside Docker..."

                    docker run --rm --name ${JMETER_CONTAINER} \
                        --network ${NETWORK} \
                        -v ${WORKSPACE}:/workspace \
                        -v ${WORKSPACE}/${RESULTS_DIR}:/results \
                        -w /workspace \
                        ${JMETER_IMAGE} \
                        -n -t "$JMX_PATH" -l /results/report.jtl
                '''
            }
        }

        stage('Archive JMeter Results') {
            steps {
                echo "📦 Archiving JMeter results..."
                archiveArtifacts artifacts: "${RESULTS_DIR}/**/*", allowEmptyArchive: true
            }
        }
    }

    post {
        always {
            echo "🧹 Cleaning up Docker containers and network..."
            sh """
                docker stop ${API_CONTAINER} || true
                docker rm ${API_CONTAINER} || true
                docker stop ${JMETER_CONTAINER} || true
                docker rm ${JMETER_CONTAINER} || true
                docker network rm ${NETWORK} || true
            """
        }
        failure {
            echo "❌ Pipeline failed!"
        }
    }
}

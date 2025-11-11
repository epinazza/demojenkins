pipeline {
    agent any

    environment {
        API_IMAGE = 'myapi-img:v1'
        API_CONTAINER = 'myapi-container'
        JMETER_CONTAINER = 'jmeter-agent'
        NETWORK = 'jenkins-net'
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
                checkout scm
            }
        }

        stage('Prepare Workspace') {
            steps {
                sh 'mkdir -p results'
            }
        }

        stage('Build API Docker Image') {
            steps {
                echo "🔧 Building WSO2 Docker image..."
                sh 'docker build -t ${API_IMAGE} -f Dockerfile .'
            }
        }

        stage('Stop & Remove Old Containers') {
            steps {
                echo "🧹 Cleaning up old containers..."
                sh '''
                docker stop ${API_CONTAINER} || true
                docker rm ${API_CONTAINER} || true
                docker stop ${JMETER_CONTAINER} || true
                docker rm ${JMETER_CONTAINER} || true
                '''
            }
        }

        stage('Run API Container') {
            steps {
                echo "🚀 Starting WSO2 Micro Integrator container..."
                sh '''
                docker network create ${NETWORK} || true
                docker run -d --name ${API_CONTAINER} --network ${NETWORK} \
                    -p 8290:8290 -p 8253:8253 ${API_IMAGE}
                '''
            }
        }

        stage('Wait for API Ready') {
            steps {
                echo "⏳ Waiting for API to be ready..."
                script {
                    retry(3) {
                        try {
                            sh '''
                            docker run --rm --network ${NETWORK} busybox sh -c \
                            "wget -qO- http://${API_CONTAINER}:8290/appointmentservices/getAppointment >/dev/null; echo $?"
                            '''
                            echo "✅ API is ready!"
                        } catch (err) {
                            echo "Attempt API not ready yet, retrying..."
                            sleep 5
                            error("API not ready")
                        }
                    }
                }
            }
        }

        stage('Debug Docker Mounts') {
            steps {
                echo "🛠️ Checking mounted files inside alpine/jmeter..."
                sh '''
                docker run --rm -v ${WORKSPACE}:/tests -v ${WORKSPACE}/results:/results \
                  alpine/jmeter sh -c "ls -l /tests && ls -l /results"
                '''
            }
        }

        stage('Run JMeter Load Test') {
            steps {
                echo "🏃 Running JMeter load test (using alpine/jmeter)..."
                sh '''
                docker run --rm --name ${JMETER_CONTAINER} --network ${NETWORK} \
                  -v ${WORKSPACE}:/tests \
                  -v ${WORKSPACE}/results:/results \
                  -w /tests \
                  alpine/jmeter \
                  -n -t API_TestPlan.jmx -l /results/report.jtl
                '''
            }
        }

        stage('Archive JMeter Report') {
            steps {
                echo "📦 Archiving JMeter report..."
                archiveArtifacts artifacts: 'results/report.jtl', fingerprint: true
            }
        }
    }

    post {
        always {
            echo "🧹 Cleaning up Docker containers..."
            sh '''
            docker stop ${API_CONTAINER} || true
            docker rm ${API_CONTAINER} || true
            docker stop ${JMETER_CONTAINER} || true
            docker rm ${JMETER_CONTAINER} || true
            '''
        }
        success {
            echo "✅ Pipeline completed successfully!"
        }
        failure {
            echo "❌ Pipeline failed!"
        }
    }
}

pipeline {
    agent any

    environment {
        // Host workspace path where Jenkins volume is mounted
        HOST_WORKSPACE = "/var/lib/docker/volumes/jenkins_home/_data/workspace/${JOB_NAME}"
    }

    stages {
        stage('Declarative: Checkout SCM') {
            steps {
                checkout scm
            }
        }

        stage('Prepare Workspace') {
            steps {
                echo "🛠 Preparing workspace..."
                sh """
                    mkdir -p ${HOST_WORKSPACE}/results
                """
            }
        }

        stage('Build WSO2 Docker Image') {
            steps {
                echo "🔧 Building WSO2 Docker image..."
                sh "docker build -t myapi-img:v1 ."
            }
        }

        stage('Stop & Remove Old WSO2 Container') {
            steps {
                echo "🧹 Cleaning up old WSO2 container (if any)..."
                sh """
                    docker stop myapi-container || true
                    docker rm myapi-container || true
                """
            }
        }

        stage('Run WSO2 Container') {
            steps {
                echo "🚀 Starting WSO2 Micro Integrator container..."
                sh """
                    docker network create jenkins-net || true
                    docker run -d --name myapi-container --network jenkins-net -p 8290:8290 -p 8253:8253 myapi-img:v1
                """
            }
        }

        stage('Wait for API Ready') {
            steps {
                echo "⏳ Waiting for API to be ready..."
                script {
                    retry(10) {
                        sh """
                            docker run --rm --network jenkins-net busybox \
                                sh -c 'wget -qO- http://myapi-container:8290/appointmentservices/getAppointment >/dev/null; echo \$?'
                        """
                        echo "✅ API is ready!"
                    }
                }
            }
        }

        stage('Prepare JMX for JMeter') {
            steps {
                echo "📄 Making JMX available for JMeter..."
                sh """
                    mkdir -p ${HOST_WORKSPACE}/results
                    cp ${WORKSPACE}/API_TestPlan.jmx ${HOST_WORKSPACE}/API_TestPlan.jmx
                    ls -l ${HOST_WORKSPACE}
                """
            }
        }

        stage('Run Load Test with JMeter') {
            steps {
                echo "🏃 Running JMeter load test..."
                sh """
                    docker run --rm --name jmeter-agent --network jenkins-net \
                        -v ${HOST_WORKSPACE}/API_TestPlan.jmx:/tests/API_TestPlan.jmx:ro \
                        -v ${HOST_WORKSPACE}/results:/tests/results:rw \
                        -w /tests justb4/jmeter:latest \
                        -n -t /tests/API_TestPlan.jmx -l /tests/results/report.jtl
                """
            }
        }

        stage('Archive JMeter Report') {
            steps {
                echo "📂 Archiving JMeter report..."
                archiveArtifacts artifacts: 'results/report.jtl', allowEmptyArchive: true
            }
        }
    }

    post {
        always {
            echo "🧹 Cleaning up Docker containers..."
            sh """
                docker stop myapi-container || true
                docker rm myapi-container || true
                docker stop jmeter-agent || true
                docker rm jmeter-agent || true
            """
        }
    }
}

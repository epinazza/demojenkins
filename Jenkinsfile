pipeline {
    agent any
    environment {
        WORKSPACE_HOST = '/var/lib/docker/volumes/jenkins_home/_data/workspace/pipelineA'
        JMETER_IMAGE = 'justb4/jmeter:latest'
        JMETER_JMX = "${WORKSPACE_HOST}/API_TestPlan.jmx"
        JMETER_RESULTS = "${WORKSPACE_HOST}/results"
        DOCKER_NETWORK = 'jenkins-net'
        WSO2_IMAGE = 'myapi-img:v1'
        WSO2_CONTAINER = 'myapi-container'
        API_URL = 'http://myapi-container:8290/appointmentservices/getAppointment'
    }
    stages {
        stage('Checkout SCM') {
            steps {
                checkout scm
            }
        }

        stage('Prepare Workspace') {
            steps {
                echo '🛠 Preparing workspace...'
                sh "mkdir -p ${JMETER_RESULTS}"
            }
        }

        stage('Build WSO2 Docker Image') {
            steps {
                echo '🔧 Building WSO2 Docker image...'
                sh "docker build -t ${WSO2_IMAGE} ."
            }
        }

        stage('Stop & Remove Old WSO2 Container') {
            steps {
                echo '🧹 Cleaning up old WSO2 container (if any)...'
                sh "docker stop ${WSO2_CONTAINER} || true"
                sh "docker rm ${WSO2_CONTAINER} || true"
            }
        }

        stage('Run WSO2 Container') {
            steps {
                echo '🚀 Starting WSO2 Micro Integrator container...'
                sh "docker network create ${DOCKER_NETWORK} || true"
                sh "docker run -d --name ${WSO2_CONTAINER} --network ${DOCKER_NETWORK} -p 8290:8290 -p 8253:8253 ${WSO2_IMAGE}"
            }
        }

        stage('Wait for API Ready') {
            steps {
                echo '⏳ Waiting for API to be ready...'
                script {
                    retry(10) {
                        def result = sh(script: "docker run --rm --network ${DOCKER_NETWORK} busybox sh -c 'wget -qO- ${API_URL} >/dev/null; echo \$?'", returnStdout: true).trim()
                        if (result != "0") {
                            echo 'Attempt API not ready yet, retrying...'
                            sleep 5
                            error('API not ready yet')
                        } else {
                            echo '✅ API is ready!'
                        }
                    }
                }
            }
        }

        stage('Check JMX File') {
            steps {
                echo '📄 Checking JMX file...'
                sh "ls -l ${JMETER_JMX}"
            }
        }

        stage('Run Load Test with JMeter') {
            steps {
                echo '🏃 Running JMeter load test...'
                sh """
                chmod -R 777 ${WORKSPACE_HOST}
                mkdir -p ${JMETER_RESULTS}
                docker run --rm --name jmeter-agent \
                    --network ${DOCKER_NETWORK} \
                    -v ${JMETER_JMX}:/tests/API_TestPlan.jmx:ro \
                    -v ${JMETER_RESULTS}:/tests/results:rw \
                    -w /tests \
                    ${JMETER_IMAGE} \
                    -n -t /tests/API_TestPlan.jmx -l /tests/results/report.jtl
                """
            }
        }

        stage('Archive JMeter Report') {
            steps {
                archiveArtifacts artifacts: 'results/**', allowEmptyArchive: true
            }
        }
    }

    post {
        always {
            echo '🧹 Cleaning up Docker containers...'
            sh "docker stop ${WSO2_CONTAINER} || true"
            sh "docker rm ${WSO2_CONTAINER} || true"
            sh "docker stop jmeter-agent || true"
            sh "docker rm jmeter-agent || true"
        }
        failure {
            echo '❌ Pipeline failed!'
        }
    }
}

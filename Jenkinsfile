pipeline {
    agent none

    environment {
        GIT_BRANCH_MAIN = 'master'
        DOCKER_REGISTRY = 'https://index.docker.io/v1/'
        DOCKER_USER = 'mrscrb'

        // Notification variables
        SLACK_CHANNEL = '#lfs261'
    }

    stages {
        stage('worker-app') {
            environment {
                APP_NAME = 'worker'
                PIPELINE_NAME = "${APP_NAME}"
                TARGET_CHANGESET = "**/${APP_NAME}/**"
                TARGET_PATH = "${APP_NAME}"
                DOCKER_REPO = "${APP_NAME}"
                DOCKER_IMAGE = "${DOCKER_USER}/${DOCKER_REPO}:v${env.BUILD_ID}"
                AGENT_IMAGE = 'maven:3.8.5-jdk-11-slim'
            }
            stages {
                stage('worker-build') {
                    agent {
                        docker {
                            image "${AGENT_IMAGE}"
                            args '-v $HOME/.m2:/root/.m2'
                        }
                    }
                    when {
                        changeset "${TARGET_CHANGESET}"
                    }
                    steps {
                        // compile the worker app
                        echo "Compiling ${APP_NAME} app.."
                        dir(path: "${TARGET_PATH}") {
                            sh 'mvn compile'
                        }
                    }
                }

                stage('worker test') {
                    agent {
                        docker {
                            image "${AGENT_IMAGE}"
                            args '-v $HOME/.m2:/root/.m2'
                        }
                    }
                    when {
                        changeset "${TARGET_CHANGESET}"
                    }
                    steps {
                        // run unit tests on worker app
                        echo "Running Unit Tets on ${APP_NAME} app."
                        dir(path: "${APP_NAME}") {
                            sh 'mvn clean test'
                        }
                    }
                }

                stage('worker-package') {
                    agent {
                        docker {
                            image "${AGENT_IMAGE}"
                            args '-v $HOME/.m2:/root/.m2'
                        }
                    }
                    when {
                        branch "${GIT_BRANCH_MAIN}" // only package on master branch
                        changeset "${TARGET_CHANGESET}"
                    }
                    steps {
                        echo "Packaging ${APP_NAME} app"
                        dir(path: "${APP_NAME}") {
                            sh 'mvn package -DskipTests'
                            archiveArtifacts(artifacts: '**/target/*.jar', fingerprint: true)
                        }
                    }
                }

                stage('worker-docker-package') {
                    agent any
                    when {
                        changeset "${TARGET_CHANGESET}"
                        branch "${GIT_BRANCH_MAIN}"
                    }
                    steps {
                        echo "Packaging ${APP_NAME} app with docker"
                        script {
                            docker.withRegistry("${DOCKER_REGISTRY}", 'dockerhub-rw') {
                                def workerImage = docker.build("${DOCKER_IMAGE}", "./${APP_NAME}")
                                workerImage.push()
                                workerImage.push("${env.BRANCH_NAME}")
                                workerImage.push('latest')
                            }
                        }
                    }
                }
            }
        }
        stage('result-app') {
            environment {
                APP_NAME = 'result'
                PIPELINE_NAME = "${APP_NAME}"
                TARGET_CHANGESET = "**/${APP_NAME}/**"
                TARGET_PATH = "${APP_NAME}"
                DOCKER_REPO = "${APP_NAME}"
                DOCKER_IMAGE = "${DOCKER_USER}/${DOCKER_REPO}:v${env.BUILD_ID}"
                AGENT_IMAGE = 'node:8.16.0-alpine'
            }
            stages {
                stage('result-build') {
                    agent {
                        docker {
                            image "${AGENT_IMAGE}"
                        }
                    }
                    when {
                        changeset "${TARGET_CHANGESET}"
                    }
                    steps {
                        echo 'Compiling result app..'
                        dir(path: "${TARGET_PATH}") {
                            sh 'npm install'
                        }
                    }
                }

                stage('result-test') {
                    agent {
                        docker {
                            image "${AGENT_IMAGE}"
                        }
                    }
                    when {
                        changeset "${TARGET_CHANGESET}"
                    }
                    steps {
                        echo 'Running Unit Tests on result app..'
                        dir(path: "${TARGET_PATH}") {
                            sh 'npm install'
                            sh 'npm test'
                        }
                    }
                }

                stage('result-docker-package') {
                    agent any
                    when {
                        changeset "${TARGET_CHANGESET}"
                        branch "${GIT_BRANCH_MAIN}"
                    }
                    steps {
                        echo 'Packaging result app with docker'
                        script {
                            docker.withRegistry("${DOCKER_REGISTRY}", 'dockerhub-rw') {
                                def resultImage = docker.build("${DOCKER_IMAGE}", "${APP_NAME}")
                                resultImage.push()
                                resultImage.push("${env.BRANCH_NAME}")
                                resultImage.push('latest')
                            }
                        }
                    }
                }
            }
        }

        stage('vote-app') {
            environment {
                APP_NAME = 'vote'
                PIPELINE_NAME = "${APP_NAME}"
                TARGET_CHANGESET = "**/${APP_NAME}/**"
                TARGET_PATH = "${APP_NAME}"
                DOCKER_REPO = "${APP_NAME}"
                DOCKER_IMAGE = "${DOCKER_USER}/${DOCKER_REPO}:v${env.BUILD_ID}"
                AGENT_IMAGE = 'python:2.7.16-slim'
            }
            stages {
                stage('vote-build') {
                    agent {
                        docker {
                            image "${AGENT_IMAGE}"
                            args '--user root'
                        }
                    }
                    when {
                        changeset "${TARGET_CHANGESET}"
                    }
                    steps {
                        echo 'Compiling vote app..'
                        dir(path: "${TARGET_PATH}") {
                            sh 'pip install -r requirements.txt'
                        }
                    }
                }
                stage('vote-test') {
                    agent {
                        docker {
                            image "${AGENT_IMAGE}"
                            args '--user root'
                        }
                    }
                    when {
                        changeset "${TARGET_CHANGESET}"
                    }
                    steps {
                        echo 'Running Unit Tests on vote app..'
                        dir(path: "${TARGET_PATH}") {
                            sh 'pip install -r requirements.txt'
                            sh 'nosetests -v'
                        }
                    }
                }
                stage('vote-docker-package') {
                    agent any
                    when {
                        changeset "${TARGET_CHANGESET}"
                        branch "${GIT_BRANCH_MAIN}"
                    }
                    steps {
                        echo "Packaging ${APP_NAME} app with docker"
                        script {
                            docker.withRegistry("${DOCKER_REGISTRY}", 'dockerhub-rw') {
                                def voteImage = docker.build("${DOCKER_IMAGE}", "${APP_NAME}")
                                voteImage.push()
                                voteImage.push("${env.BRANCH_NAME}")
                                voteImage.push('latest')
                            }
                        }
                    }
                }
            }
        }

        stage('deploy to dev') {
            agent any
            when {
                branch "${GIT_BRANCH_MAIN}"
            }
            steps {
                echo 'Deploy instavote app with docker compose'
                sh 'docker-compose up -d'
            }
        }
    }

    post {
        always {
            echo 'Building mono pipeline for voting app is completed.'
        }
        success {
            slackSend(channel: "${SLACK_CHANNEL}", message: "Build Success: ${env.JOB_NAME} ${env.BUILD_NUMBER}", iconEmoji: 'white_check_mark')
        }
        failure {
            slackSend(channel: "${SLACK_CHANNEL}", message: "Build Failed: ${env.JOB_NAME} ${env.BUILD_NUMBER}", iconEmoji: 'x')
        }
    }
}

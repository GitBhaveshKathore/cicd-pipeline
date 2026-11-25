pipeline {
    agent any

    environment {
        MINIKUBE_HOME = '/home/ec2-user/.minikube'
        KUBECONFIG = '/home/ec2-user/.kube/config'
        DOCKER_IMAGE = 'kubebhavesh/flask-cicd-app'
        VENV = "${WORKSPACE}/venv"
    }

    triggers {
        githubPush()
    }

    stages {

        stage('Checkout') {
            steps {
                git(
                    branch: 'main',
                    url: 'https://github.com/GitBhaveshKathore/cicd-pipeline.git'
                )
            }
        }

        stage('Setup Python Environment') {
            steps {
                sh '''
                python3 -m venv venv

                . venv/bin/activate

                pip install --upgrade pip

                pip install -r requirements.txt
                '''
            }
        }

        stage('Unit Tests') {
            steps {
                sh '''
                . venv/bin/activate

                pytest tests/ -v || true
                '''
            }
        }

        stage('Build Validation') {
            steps {
                sh '''
                . venv/bin/activate

                python -c "import app; print('Build OK')"
                '''
            }
        }

        stage('Docker Build') {
            steps {
                sh """
                docker build -t ${DOCKER_IMAGE}:${BUILD_NUMBER} .
                docker tag ${DOCKER_IMAGE}:${BUILD_NUMBER} ${DOCKER_IMAGE}:latest
                """
            }
        }

        stage('DockerHub Login & Push') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-creds',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )
                ]) {
           
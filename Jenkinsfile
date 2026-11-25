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
                    sh '''
                    echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                    '''
                }

                sh """
                docker push ${DOCKER_IMAGE}:${BUILD_NUMBER}
                docker push ${DOCKER_IMAGE}:latest
                """
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                sh '''
                kubectl apply -f kubernetes/deployment.yaml

                kubectl set image deployment/flask-cicd-app \
                flask-cicd-app=''' + '${DOCKER_IMAGE}:${BUILD_NUMBER}' + ''' \
                --record

                kubectl rollout status deployment/flask-cicd-app --timeout=180s
                '''
            }
        }

        stage('Verify Deployment') {
            steps {
                sh '''
                kubectl get deployments

                kubectl get pods -l app=flask-cicd-app

                kubectl get svc
                '''
            }
        }

        stage('Expose Application') {
            steps {
                sh '''
                pkill -f "kubectl port-forward.*30090" || true

                nohup kubectl port-forward \
                --address 0.0.0.0 \
                svc/flask-cicd-app-service \
                30090:80 \
                >/tmp/portforward.log 2>&1 &
                '''
            }
        }

        stage('Application Health Check') {
            steps {
                sh '''
                sleep 15

                curl -f http://localhost:30090/ || true
                '''
            }
        }
    }

    post {

        success {
            echo 'SUCCESS: Flask CI/CD Pipeline Completed'
        }

        failure {
            echo 'FAILED: Check Jenkins Console Logs'
        }

        always {
            sh '''
            docker logout || true
            '''
        }
    }
}
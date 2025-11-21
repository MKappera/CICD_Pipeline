pipeline {
    agent any

    environment {
        PYTHON = "/usr/bin/python3"
        VENV = "venv"

        DEPLOY_USER = "ubuntu"
        DEPLOY_IP = "54.90.73.208"       // <-- Replace with your Deployment Server Public IP
        DEPLOY_PATH = "/opt/apps"
    }

    stages {

        stage('Checkout Code') {
            steps {
                echo "Pulling code from GitHub..."
                checkout scm
            }
        }

        stage('Setup Python Virtual Environment') {
            steps {
                echo "Creating virtual environment..."
                sh """
                ${PYTHON} -m venv ${VENV}
                """
            }
        }

        stage('Install Dependencies') {
            steps {
                echo "Installing requirements..."
                sh """
                . ${VENV}/bin/activate
                pip install --upgrade pip
                pip install -r requirements.txt
                """
            }
        }

        stage('Run Tests') {
            steps {
                echo "Running Pytest..."
                sh """
                . ${VENV}/bin/activate
                export PYTHONPATH=\$(pwd)
                pytest --junitxml=results.xml
                """
            }
            post {
                always {
                    junit 'results.xml'
                }
            }
        }

        stage('Package Application') {
            steps {
                echo "Packaging application..."
                sh """
                rm -rf package
                mkdir -p package
                cp -r app package/
                tar -czf sample-app.tar.gz -C package app
                """
                archiveArtifacts artifacts: 'sample-app.tar.gz', fingerprint: true
            }
        }

        stage('Deploy to Server') {
            steps {
                echo "Deploying to EC2 Deployment Server..."
                withCredentials([sshUserPrivateKey(credentialsId: 'deploy-key', keyFileVariable: 'SSH_KEY')]) {
                    sh """
                    scp -o StrictHostKeyChecking=no -i $SSH_KEY sample-app.tar.gz ${DEPLOY_USER}@${DEPLOY_IP}:${DEPLOY_PATH}/
                    """
                }
            }
        }

        stage('Extract on Deployment Server') {
            steps {
                echo "Extracting files on remote server..."
                withCredentials([sshUserPrivateKey(credentialsId: 'deploy-key', keyFileVariable: 'SSH_KEY')]) {
                    sh """
                    ssh -o StrictHostKeyChecking=no -i $SSH_KEY ${DEPLOY_USER}@${DEPLOY_IP} "tar -xzf ${DEPLOY_PATH}/sample-app.tar.gz -C ${DEPLOY_PATH}/"
                    """
                }
            }
        }

        stage('Restart Application Service') {
            steps {
                echo "Restarting Python app service..."
                withCredentials([sshUserPrivateKey(credentialsId: 'deploy-key', keyFileVariable: 'SSH_KEY')]) {
                    sh """
                    ssh -o StrictHostKeyChecking=no -i $SSH_KEY ${DEPLOY_USER}@${DEPLOY_IP} "sudo systemctl restart pythonapp"
                    """
                }
            }
        }
    }
}

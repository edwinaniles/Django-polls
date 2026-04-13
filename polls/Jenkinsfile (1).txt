pipeline {
    agent any

    environment {
        DOCKER_IMAGE = 'kbhola001/django-project' // (MODIFY: Enter your Docker Hub username and image name) Example: 'yourusername/your-image-name'
        EC2_USER = 'ubuntu'  // (MODIFY) Change to 'ubuntu' if using an Ubuntu AMI or 'ec2-user' if using Amazon Linux AMI
        EC2_HOST = "3.238.183.57" //(MODIFY: Enter your EC2 instance public IP address)
        EC2_KEY = credentials('ec2-ssh-private-key')  // (MODIFY: ensure you create this credential in Jenkins. This is the SSH private key of your EC2 instance)
        DOCKER_CREDS = 'docker-hub-credentials' // Set up Jenkins credentials for Docker Hub. Ensure the ID is 'docker-hub-credentials'
        PROJECT_DIR = "/home/ubuntu/pythonprojects/django_polls"  // (MODIFY: enter the path to your Django project on the EC2 instance)
    }

    //triggers {
      //  githubPush()
    //}

    stages {
        stage('Clone Repository') {
            steps {
                // (MODIFY: Enter your GitHub repository URL)
                git branch: 'main', url: 'https://github.com/kestonbhola/django_polls.git'
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    docker.build("${DOCKER_IMAGE}:latest")
                }
            }
        }

        stage('Push to Docker Hub') {
            steps {
                script {
                    docker.withRegistry('https://index.docker.io/v1/', DOCKER_CREDS) {
                        docker.image("${DOCKER_IMAGE}:latest").push()
                        echo "Image pushed to Docker Hub"
                    }
                }
            }
        }

        stage('Deploy on EC2') {
            steps {
                script {
                    sshagent (credentials: ['ec2-ssh-private-key']) {
                        sh """
                        ssh -o StrictHostKeyChecking=no ${EC2_USER}@${EC2_HOST} '
                            docker pull ${DOCKER_IMAGE}:latest
                            docker ps -a -q -f name=django-container | grep -q . && docker stop django-container || true
                            docker ps -a -q -f name=django-container | grep -q . && docker rm django-container || true
                            docker run -d --name django-container -p 80:80 ${DOCKER_IMAGE}:latest
                            sleep 5
                            docker ps -a
                            docker logs django-container || true
                        '
                        """
                    }
                }
            }
        }
    }

    post {
        success {
            echo 'Deployment Successful!'
        }
        failure {
            echo 'Deployment Failed.'
        }
    }
}
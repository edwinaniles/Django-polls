pipeline {
    agent any

    triggers {
        githubPush()
    }

    environment {
        SITE_NAME  = "Django-polls"
        WEB_ROOT   = "/var/www/Django-polls"
        NGINX_CONF = "/etc/nginx/sites-available/Django-polls"
    }

    options {
        timestamps()
    }

    stages {

        stage('Clean Workspace') {
            steps {
                deleteDir()
            }
        }

        stage('Checkout') {
            steps {
                git url: 'https://github.com/edwinaniles/Django-polls', branch: 'main'
                sh 'ls -la'
            }
        }

        stage('Verify Project Files') {
            steps {
                sh '''
                    set -e
                    pwd
                    ls -la
                    ls -l requirements.txt || true

                    test -f manage.py
                    test -d polls
                    test -f requirements.txt
                '''
            }
        }

        stage('Install Nginx and Python') {
            steps {
                sh '''
                    set -e
                    sudo apt update
                    sudo apt install -y nginx python3 python3-pip python3-venv
                '''
            }
        }

        stage('Create Web Root') {
            steps {
                sh '''
                    set -e
                    sudo mkdir -p "$WEB_ROOT"
                    sudo chown -R jenkins:jenkins "$WEB_ROOT"
                '''
            }
        }

        stage('Deploy Project Files') {
            steps {
                sh '''
                    set -e
                    rm -rf "$WEB_ROOT"/*
                    sudo cp -r * "$WEB_ROOT"/
                '''
            }
        }

        stage('Set Up Virtual Environment') {
            steps {
                sh '''
                    set -e
                    cd "$WEB_ROOT"

                    rm -rf venv
                    python3 -m venv venv

                    . venv/bin/activate
                    pip install --upgrade pip
                    pip install -r requirements.txt
                '''
            }
        }

        stage('Run Django Setup') {
            steps {
                sh '''
                    set -e
                    cd "$WEB_ROOT"

                    . venv/bin/activate
                    python manage.py migrate
                    python manage.py collectstatic --noinput || true
                '''
            }
        }

        stage('Start Django Server') {
            steps {
                sh '''
                    set -e
                    cd "$WEB_ROOT"

                    pkill -f "manage.py runserver" || true

                    nohup venv/bin/python manage.py runserver 0.0.0.0:8000 --noreload > django.log 2>&1 &

                    sleep 5
                    curl http://127.0.0.1:8000 || true
                '''
            }
        }

        stage('Configure Nginx Site') {
            steps {
                sh '''
                    set -e

                    sudo tee "$NGINX_CONF" > /dev/null <<EOF
server {
    listen 80;
    server_name _;

    location /static/ {
        alias /var/www/Django-polls/static/;
    }

    location / {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
EOF

                    sudo ln -sf "$NGINX_CONF" /etc/nginx/sites-enabled/Django-polls
                    sudo rm -f /etc/nginx/sites-enabled/default || true
                    sudo nginx -t
                '''
            }
        }

        stage('Start Nginx') {
            steps {
                sh '''
                    set -e
                    sudo systemctl enable nginx
                    sudo systemctl restart nginx
                '''
            }
        }

        stage('Test Website Locally') {
            steps {
                sh '''
                    set -e
                    curl -I http://127.0.0.1 || true
                '''
            }
        }
    }

    post {
        success {
            echo 'Deployment successful.'
        }
        failure {
            echo 'Deployment failed. Check Jenkins console output.'
        }
    }
}
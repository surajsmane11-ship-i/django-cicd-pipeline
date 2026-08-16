pipeline {
    agent any

    environment {
        PROD_HOST = "13.126.111.217"
        PROD_USER = "ubuntu"
        PROD_DIR  = "/home/ubuntu/django-cicd-project"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
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

        stage('Run Django Tests') {
            steps {
                sh '''
                    . venv/bin/activate

                    python3 manage.py test
                '''
            }
        }

        stage('Deploy to Production') {
            steps {

                sshagent(credentials: ['ec2-ssh']) {

                    sh '''
                        echo "======================================"
                        echo "Connecting to Production Server"
                        echo "======================================"

                        ssh -o StrictHostKeyChecking=no ${PROD_USER}@${PROD_HOST} "bash -s" << 'REMOTE_SCRIPT'

                        set -e

                        echo "======================================"
                        echo "Connected to Production Server"
                        echo "======================================"

                        cd /home/ubuntu/django-cicd-project

                        echo "======================================"
                        echo "CURRENT COMMIT"
                        echo "======================================"

                        git log --oneline -1

                        echo "======================================"
                        echo "FETCHING LATEST CODE"
                        echo "======================================"

                        git fetch origin

                        echo "======================================"
                        echo "UPDATING PRODUCTION CODE"
                        echo "======================================"

                        git reset --hard origin/main

                        echo "======================================"
                        echo "UPDATED COMMIT"
                        echo "======================================"

                        git log --oneline -1

                        echo "======================================"
                        echo "CREATING VIRTUAL ENVIRONMENT"
                        echo "======================================"

                        if [ ! -d "venv" ]; then
                            python3 -m venv venv
                        fi

                        source venv/bin/activate

                        echo "======================================"
                        echo "INSTALLING REQUIREMENTS"
                        echo "======================================"

                        pip install -r requirements.txt

                        echo "======================================"
                        echo "RUNNING MIGRATIONS"
                        echo "======================================"

                        python3 manage.py migrate

                        echo "======================================"
                        echo "COLLECTING STATIC FILES"
                        echo "======================================"

                        python3 manage.py collectstatic --noinput || true

                        echo "======================================"
                        echo "STOPPING OLD DJANGO SERVER"
                        echo "======================================"

                        pkill -f "manage.py runserver" || true

                        sleep 2

                        echo "======================================"
                        echo "STARTING DJANGO SERVER"
                        echo "======================================"

                        nohup python3 manage.py runserver 0.0.0.0:8000 > django.log 2>&1 &

                        sleep 5

                        echo "======================================"
                        echo "VERIFYING DJANGO SERVER"
                        echo "======================================"

                        pgrep -f "manage.py runserver"

                        echo "======================================"
                        echo "DEPLOYMENT SUCCESSFUL"
                        echo "======================================"

                        REMOTE_SCRIPT
                    '''
                }
            }
        }
    }

    post {

        success {
            echo "CI/CD Pipeline Completed Successfully"
        }

        failure {
            echo "CI/CD Pipeline Failed"
        }
    }
}
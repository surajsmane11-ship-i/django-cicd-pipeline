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
                    set -e

                    python3 -m venv venv
                    . venv/bin/activate

                    python -m pip install --upgrade pip
                    python -m pip install -r requirements.txt
                '''
            }
        }

        stage('Run Django Tests') {
            steps {
                sh '''
                    set -e

                    . venv/bin/activate

                    python manage.py test
                '''
            }
        }

        stage('Deploy to Production') {
            steps {

                sshagent(credentials: ['ec2-ssh']) {

                    sh '''
                        set -e

                        echo "======================================"
                        echo "Connecting to Production Server"
                        echo "======================================"

                        ssh -o StrictHostKeyChecking=no \
                            ${PROD_USER}@${PROD_HOST} 'bash -s' <<'REMOTE_SCRIPT'

set -e

PROD_DIR="/home/ubuntu/django-cicd-project"

echo "======================================"
echo "Connected to Production Server"
echo "======================================"

cd "$PROD_DIR"

echo "======================================"
echo "CURRENT COMMIT"
echo "======================================"

git log --oneline -1 || true

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

python -m pip install --upgrade pip
python -m pip install -r requirements.txt

echo "======================================"
echo "RUNNING MIGRATIONS"
echo "======================================"

python manage.py migrate

echo "======================================"
echo "COLLECTING STATIC FILES"
echo "======================================"

python manage.py collectstatic --noinput

echo "======================================"
echo "STOPPING OLD DJANGO SERVER"
echo "======================================"

if [ -f django.pid ]; then

    PID=$(cat django.pid)

    if kill -0 "$PID" 2>/dev/null; then
        echo "Stopping Django process: $PID"

        kill "$PID" || true

        sleep 3

        if kill -0 "$PID" 2>/dev/null; then
            echo "Force stopping Django process: $PID"
            kill -9 "$PID" || true
        fi
    fi

    rm -f django.pid
fi

echo "======================================"
echo "STARTING DJANGO SERVER"
echo "======================================"

nohup python manage.py runserver 0.0.0.0:8000 \
    > django.log 2>&1 &

DJANGO_PID=$!

echo "$DJANGO_PID" > django.pid

echo "Django PID: $DJANGO_PID"

sleep 5

echo "======================================"
echo "VERIFYING DJANGO SERVER"
echo "======================================"

if kill -0 "$DJANGO_PID" 2>/dev/null; then

    echo "Django server is running"

else

    echo "Django server FAILED to start"

    echo "======================================"
    echo "DJANGO LOG"
    echo "======================================"

    cat django.log

    rm -f django.pid

    exit 1
fi

echo "======================================"
echo "DEPLOYMENT SUCCESSFUL"
echo "======================================"

exit 0
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
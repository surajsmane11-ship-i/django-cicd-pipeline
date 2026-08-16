pipeline {
    agent any

    environment {
        PROD_HOST = " 13.126.111.217 "
        PROD_USER = "ubuntu"
        PROD_DIR  = "/home/ubuntu/django-cicd-project_new"
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
                    sh """
                    ssh -o StrictHostKeyChecking=no ${PROD_USER}@${PROD_HOST} << EOF
set -e

cd ${PROD_DIR}

echo "========== CURRENT COMMIT =========="
git log --oneline -1

echo "========== FETCH LATEST CODE =========="
git fetch origin
git reset --hard origin/main

echo "========== UPDATED COMMIT =========="
git log --oneline -1

if [ ! -d venv ]; then
    python3 -m venv venv
fi

source venv/bin/activate

echo "========== INSTALLING REQUIREMENTS =========="
pip install -r requirements.txt

echo "========== RUNNING MIGRATIONS =========="
python3 manage.py migrate

echo "========== COLLECT STATIC =========="
python3 manage.py collectstatic --noinput || true

echo "========== STOPPING OLD DJANGO SERVER =========="
pkill -f "manage.py runserver" || true

sleep 2

echo "========== STARTING DJANGO SERVER =========="
nohup python3 manage.py runserver 0.0.0.0:8000 > django.log 2>&1 &

sleep 5

echo "========== VERIFY RUNSERVER =========="
pgrep -f "manage.py runserver"

echo "========== DEPLOYMENT SUCCESSFUL =========="
EOF
                    """
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
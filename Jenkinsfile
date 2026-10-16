
pipeline {
    agent any

    environment {
        IMAGE_NAME = 'uptime-kuma:dev'
        REMOTE_HOST = 'ubuntu@172.31.19.216'
        SSH_KEY = '/var/lib/jenkins/.ssh/id_ed25519'
        REMOTE_TAR = '/home/ubuntu/uptime-kuma-dev.tar'
        CONTAINER_NAME = 'uptime-kuma'
    }

    stages {
        stage('Checkout / Verify') {
            steps {
                sh '''
                    echo "Checking source code..."
                    pwd
                    ls -la
                    test -f docker/dockerfile
                '''
            }
        }

        
        stage('Build Frontend') {
            steps {
                sh '''
                    npm ci
                    npm run build
                    test -f dist/index.html
                '''
            }
        }


        stage('Build Docker Image') {
            steps {
                sh '''
                    docker build \
                      --target release \
                      -f docker/dockerfile \
                      -t ${IMAGE_NAME} .
                '''
            }
        }

        stage('Export Docker Image') {
            steps {
                sh '''
                    docker save ${IMAGE_NAME} -o /tmp/uptime-kuma-dev.tar
                    chmod 644 /tmp/uptime-kuma-dev.tar
                    ls -lh /tmp/uptime-kuma-dev.tar
                '''
            }
        }

        stage('Transfer Image to Gaint') {
            steps {
                sh '''
                    scp \
                      -i ${SSH_KEY} \
                      -o IdentitiesOnly=yes \
                      -o StrictHostKeyChecking=accept-new \
                      /tmp/uptime-kuma-dev.tar \
                      ${REMOTE_HOST}:${REMOTE_TAR}
                '''
            }
        }

        stage('Deploy on Gaint') {
            steps {
                sh '''
                    ssh \
                      -i ${SSH_KEY} \
                      -o IdentitiesOnly=yes \
                      -o StrictHostKeyChecking=accept-new \
                      ${REMOTE_HOST} 'bash -s' <<'REMOTE_SCRIPT'
set -eu

echo "Loading Docker image..."
sudo docker load -i /home/ubuntu/uptime-kuma-dev.tar

echo "Removing old container if it exists..."
sudo docker rm -f uptime-kuma 2>/dev/null || true

echo "Starting new container..."
sudo docker run -d \
  --name uptime-kuma \
  --restart unless-stopped \
  -p 3001:3001 \
  -v uptime-kuma-data:/app/data \
  uptime-kuma:dev

echo "Waiting for application startup..."
sleep 15

if ! sudo docker inspect uptime-kuma >/dev/null 2>&1; then
    echo "ERROR: Container was not created."
    exit 1
fi

if [ "$(sudo docker inspect -f '{{.State.Running}}' uptime-kuma)" != "true" ]; then
    echo "ERROR: Container is not running."
    sudo docker logs --tail 100 uptime-kuma
    exit 1
fi

echo "Container is running."
sudo docker ps --filter name=uptime-kuma

echo "Checking application response..."
for i in $(seq 1 12); do
    if curl -fsS http://127.0.0.1:3001/ >/dev/null; then
        echo "SUCCESS: Uptime Kuma is responding on port 3001."
        exit 0
    fi
    sleep 5
done

echo "ERROR: Application did not respond on port 3001."
sudo docker logs --tail 100 uptime-kuma
exit 1
REMOTE_SCRIPT
                '''
            }
        }
    }

    post {
        always {
            sh 'rm -f /tmp/uptime-kuma-dev.tar'
        }
        success {
            echo 'SUCCESS: Uptime Kuma built and deployed on Gaint!'
        }
        failure {
            echo 'FAILED: Check the stage logs in Console Output.'
        }
    }
}


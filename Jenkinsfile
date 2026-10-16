pipeline {
    agent any

    environment {
        IMAGE_NAME = 'uptime-kuma:dev'
        REMOTE_HOST = 'ubuntu@172.31.19.216'
        SSH_KEY = '/var/lib/jenkins/.ssh/id_ed25519'
        REMOTE_TAR = '/home/ubuntu/uptime-kuma-dev.tar'
    }

    stages {
        stage('Checkout / Verify') {
            steps {
                sh 'pwd'
                sh 'ls -l'
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

        stage('Export Image') {
            steps {
                sh '''
                    docker save ${IMAGE_NAME} -o /tmp/uptime-kuma-dev.tar
                    chmod 644 /tmp/uptime-kuma-dev.tar
                '''
            }
        }

        stage('Copy Image to Gaint') {
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
set -e

sudo docker load -i /home/ubuntu/uptime-kuma-dev.tar

# Preserve existing application data volume.
sudo docker stop uptime-kuma || true
sudo docker rm uptime-kuma || true


sudo docker run -d --name uptime-kuma --restart unless-stopped -p 3001:3001 -v uptime-kuma-data:/app/data uptime-kuma:dev


# Wait for the application to become healthy.
for i in $(seq 1 30); do
    STATUS=$(sudo docker inspect 
      --format='{{.State.Health.Status}}' uptime-kuma 2>/dev/null || true)

    if [ "$STATUS" = "healthy" ]; then
        echo "Uptime Kuma is healthy."
        sudo docker ps --filter name=uptime-kuma
        exit 0
    fi

    sleep 10
done

echo "Health check timed out. Container logs:"
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
            echo 'Deployment completed successfully!'
        }
        failure {
            echo 'Pipeline failed. Check the stage logs.'
        }
    }
}

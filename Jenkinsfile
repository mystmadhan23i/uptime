

pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
        timeout(time: 90, unit: 'MINUTES')
    }

    environment {
        IMAGE_NAME = 'uptime-kuma:dev'
        GAIN_HOST  = '172.31.19.216'
        GAIN_USER  = 'ubuntu'
        SSH_KEY    = '/var/lib/jenkins/.ssh/id_ed25519'
        TAR_FILE   = "${WORKSPACE}/uptime-kuma-dev.tar"
    }

    stages {
        stage('Checkout / Verify') {
            steps {
                sh '''
                    set -eu
                    echo "Checking source code..."
                    test -f docker/dockerfile
                    test -f package.json
                    test -f package-lock.json
                    test -f Jenkinsfile
                    echo "Source files verified."
                '''
            }
        }

        stage('Build Frontend') {
            steps {
                sh '''
                    set -eu

                    echo "Installing frontend dependencies in Docker..."
                    docker run --rm \
                      -v "$WORKSPACE:/app" \
                      -w /app \
                      --entrypoint npm \
                      louislam/uptime-kuma:base3 ci

                    echo "Building frontend..."
                    docker run --rm \
                      -v "$WORKSPACE:/app" \
                      -w /app \
                      --entrypoint npm \
                      louislam/uptime-kuma:base3 run build

                    test -s "$WORKSPACE/dist/index.html"
                    echo "Frontend build verified."
                '''
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    set -eu

                    docker build \
                      --target release \
                      -f docker/dockerfile \
                      -t "$IMAGE_NAME" .

                    echo "Verifying frontend exists inside the image..."
                    docker run --rm \
                      --entrypoint sh \
                      "$IMAGE_NAME" \
                      -c 'test -s /app/dist/index.html'

                    echo "Docker image verified."
                '''
            }
        }

        stage('Export Docker Image') {
            steps {
                sh '''
                    set -eu
                    rm -f "$TAR_FILE"
                    docker save "$IMAGE_NAME" -o "$TAR_FILE"
                    test -s "$TAR_FILE"
                    ls -lh "$TAR_FILE"
                '''
            }
        }

        stage('Transfer Image to Gaint') {
            steps {
                sh '''
                    set -eu
                    chmod 600 "$SSH_KEY"

                    ssh -i "$SSH_KEY" \
                      -o IdentitiesOnly=yes \
                      -o StrictHostKeyChecking=accept-new \
                      "$GAIN_USER@$GAIN_HOST" \
                      'mkdir -p /home/ubuntu/uptime-deploy'

                    scp -i "$SSH_KEY" \
                      -o IdentitiesOnly=yes \
                      -o StrictHostKeyChecking=accept-new \
                      "$TAR_FILE" \
                      "$GAIN_USER@$GAIN_HOST:/home/ubuntu/uptime-deploy/uptime-kuma-dev.tar"

                    echo "Image transferred."
                '''
            }
        }

        stage('Deploy on Gaint') {
            steps {
                sh '''
                    set -eu

                    ssh -i "$SSH_KEY" \
                      -o IdentitiesOnly=yes \
                      -o StrictHostKeyChecking=accept-new \
                      "$GAIN_USER@$GAIN_HOST" 'bash -s' <<'REMOTE'
set -eu

IMAGE_TAR="/home/ubuntu/uptime-deploy/uptime-kuma-dev.tar"

echo "Loading Docker image..."
sudo docker load -i "$IMAGE_TAR"

echo "Ensuring persistent data volume exists..."
sudo docker volume create uptime-kuma-data >/dev/null

echo "Preserving the existing /app/data volume if one is attached..."
DATA_VOLUME="uptime-kuma-data"

if sudo docker container inspect uptime-kuma >/dev/null 2>&1; then
    EXISTING_VOLUME=$(sudo docker inspect \
      --format '{{range .Mounts}}{{if eq .Destination "/app/data"}}{{.Name}}{{end}}{{end}}' \
      uptime-kuma)

    if [ -n "$EXISTING_VOLUME" ]; then
        DATA_VOLUME="$EXISTING_VOLUME"
        echo "Using existing data volume: $DATA_VOLUME"
    fi

    echo "Stopping old container..."
    sudo docker stop uptime-kuma || true
    sudo docker rm uptime-kuma
fi

echo "Starting updated container..."
sudo docker run -d \
  --name uptime-kuma \
  --restart unless-stopped \
  -p 3001:3001 \
  -v "$DATA_VOLUME:/app/data" \
  uptime-kuma:dev

echo "Waiting for the application to start..."
READY=0

for i in $(seq 1 36); do
    if sudo docker exec uptime-kuma \
        node -e "fetch('http://127.0.0.1:3001/').then(r=>process.exit(r.ok?0:1)).catch(()=>process.exit(1))"
    then
        READY=1
        break
    fi

    if ! sudo docker inspect \
        --format '{{.State.Running}}' uptime-kuma | grep -q true; then
        echo "Container stopped unexpectedly."
        sudo docker logs --tail 100 uptime-kuma
        exit 1
    fi

    sleep 5
done

if [ "$READY" -ne 1 ]; then
    echo "Application did not become ready."
    sudo docker logs --tail 100 uptime-kuma
    exit 1
fi

echo "Deployment verified."
sudo docker ps --filter name=uptime-kuma
REMOTE
                '''
            }
        }
    }

    post {
        success {
            echo 'SUCCESS: Image built, transferred, deployed, and HTTP readiness verified.'
        }

        failure {
            echo 'FAILED: Check the first failing stage in Console Output.'
        }

        always {
            sh 'rm -f "$TAR_FILE"'
        }
    }
}



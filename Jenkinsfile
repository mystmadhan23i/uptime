pipeline {
    agent any

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
                      -t uptime-kuma:dev .
                '''
            }
        }

        stage('Export Image') {
            steps {
                sh '''
                    docker save uptime-kuma:dev -o /tmp/uptime-kuma-dev.tar
                    chmod 644 /tmp/uptime-kuma-dev.tar
                '''
            }
        }

        stage('Copy Image to Gaint') {
            steps {
                sh '''
                    scp \
                      -i /var/lib/jenkins/.ssh/id_ed25519 \
                      -o IdentitiesOnly=yes \
                      -o StrictHostKeyChecking=accept-new \
                      /tmp/uptime-kuma-dev.tar \
                      ubuntu@172.31.19.216:/home/ubuntu/
                '''
            }
        }
    }
}

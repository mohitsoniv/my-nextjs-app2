pipeline {
    agent any

    tools {
        nodejs 'node18'
    }

    environment {
        EC2_USER = 'ubuntu'
        EC2_IP = '13.222.38.219'
        EC2_HOST = "${EC2_USER}@${EC2_IP}"
        SSH_KEY_ID = 'ec2-ssh-key'
        DEPLOY_DIR = '/var/www/myapp'
        NEXT_PORT = '3000'
    }

    stages {

        stage('Checkout Code') {
            steps {
                git 'https://github.com/mohitsoniv/my-nextjs-app2.git'
            }
        }

        stage('Verify Node & NPM') {
            steps {
                sh '''
                    echo "🔧 Verifying Node.js and npm"
                    echo "Node: $(which node)"
                    node -v
                    npm -v
                '''
            }
        }

        stage('Install All Dependencies') {
            steps {
                sh '''
                    echo "🧹 Cleaning old modules"
                    rm -rf node_modules package-lock.json || true
                    npm cache clean --force
                    echo "📦 Installing ALL dependencies (incl. dev)"
                    npm install
                '''

pipeline {
    agent any

    tools {
        nodejs 'node18' // Jenkins Global Tool Configuration name
    }

    environment {
        EC2_USER = 'ubuntu'
        EC2_IP = '13.222.38.219'
        EC2_HOST = "${EC2_USER}@${EC2_IP}"
        SSH_KEY_ID = 'ec2-ssh-key' // Jenkins credentials ID
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

        stage('Install Dependencies') {
            steps {
                sh '''
                    echo "🧹 Cleaning previous installs"
                    rm -rf node_modules package-lock.json || true
                    npm cache clean --force
                    echo "📦 Installing dependencies"
                    npm install --legacy-peer-deps
                '''
            }
        }

        stage('Build App') {
            steps {
                sh '''
                    echo "🏗️ Building the Next.js app"
                    npm run build
                '''
            }
        }

        stage('Package Standalone Build') {
            steps {
                sh '''
                    echo "📦 Packaging standalone build"
                    mkdir -p packaged-app
                    cp -r .next/standalone .next/static public package.json next.config.* packaged-app/ 2>/dev/null || echo "Some optional files not found"
                '''
            }
        }

        stage('Add EC2 Host to known_hosts') {
            steps {
                sshagent(credentials: ["${SSH_KEY_ID}"]) {
                    sh '''
                        echo "🔐 Adding EC2 to known_hosts"
                        mkdir -p ~/.ssh
                        ssh-keyscan -H ${EC2_IP} >> ~/.ssh/known_hosts
                        chmod 644 ~/.ssh/known_hosts
                    '''
                }
            }
        }

        stage('Install Node.js on EC2') {
            steps {
                sshagent(credentials: ["${SSH_KEY_ID}"]) {
                    sh '''
                        echo "🛠️ Installing Node.js on EC2 (if not installed)"
                        ssh ${EC2_HOST} "if ! command -v node >/dev/null 2>&1; then curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash - && sudo apt-get install -y nodejs; fi"
                    '''
                }
            }
        }

        stage('Deploy to EC2') {
            steps {
                sshagent(credentials: ["${SSH_KEY_ID}"]) {
                    sh '''
                        echo "📤 Deploying to EC2"
                        ssh ${EC2_HOST} "sudo mkdir -p ${DEPLOY_DIR} && sudo chown -R ${EC2_USER}:${EC2_USER} ${DEPLOY_DIR}"
                        scp -r packaged-app/* ${EC2_HOST}:${DEPLOY_DIR}
                    '''
                }
            }
        }

        stage('Start Next.js App') {
            steps {
                sshagent(credentials: ["${SSH_KEY_ID}"]) {
                    sh '''
                        echo "🚀 Starting Next.js app on EC2"
                        ssh ${EC2_HOST} "cd ${DEPLOY_DIR} && npm install --omit=dev && fuser -k ${NEXT_PORT}/tcp || true && nohup npx next start -p ${NEXT_PORT} > out.log 2>&1 &"
                    '''
                }
            }
        }
    }

    post {
        success {
            echo '✅ Build and deployment successful!'
        }
        failure {
            echo '❌ Deployment failed. Check the logs above for details.'
        }
    }
}

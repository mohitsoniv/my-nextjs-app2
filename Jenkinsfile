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

        stage('Package App for Deployment') {
            steps {
                sh '''
                    echo "📦 Packaging app"
                    rm -rf packaged-app
                    mkdir -p packaged-app

                    cp -r .next public package.json next.config.* tailwind.config.* postcss.config.* packaged-app/

                    if [ -d "app" ]; then cp -r app packaged-app/; fi
                    if [ -d "pages" ]; then cp -r pages packaged-app/; fi
                '''
            }
        }

        stage('Add EC2 to known_hosts') {
            steps {
                sshagent(credentials: ["${SSH_KEY_ID}"]) {
                    sh '''
                        echo "🔐 Trusting EC2 host key"
                        mkdir -p ~/.ssh
                        ssh-keyscan -H ${EC2_IP} >> ~/.ssh/known_hosts
                        chmod 644 ~/.ssh/known_hosts
                    '''
                }
            }
        }

        stage('Ensure Node.js on EC2') {
            steps {
                sshagent(credentials: ["${SSH_KEY_ID}"]) {
                    sh '''
                        echo "🛠️ Installing Node.js on EC2 if missing"
                        ssh ${EC2_HOST} '
                          if ! command -v node >/dev/null 2>&1; then
                            curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
                            sudo apt-get install -y nodejs
                          fi
                        '
                    '''
                }
            }
        }

        stage('Deploy to EC2') {
            steps {
                sshagent(credentials: ["${SSH_KEY_ID}"]) {
                    sh '''
                        echo "📤 Deploying files to EC2"
                        ssh ${EC2_HOST} "sudo mkdir -p ${DEPLOY_DIR} && sudo chown -R ${EC2_USER}:${EC2_USER} ${DEPLOY_DIR}"
                        scp -r packaged-app/* ${EC2_HOST}:${DEPLOY_DIR}
                    '''
                }
            }
        }

        stage('Start App on EC2') {
            steps {
                sshagent(credentials: ["${SSH_KEY_ID}"]) {
                    sh '''
                        echo "🚀 Running app on EC2"
                        ssh ${EC2_HOST} '
                          cd ${DEPLOY_DIR}
                          npm install
                          npm run build
                          fuser -k ${NEXT_PORT}/tcp || true
                          nohup npx next start -p ${NEXT_PORT} -H 0.0.0.0 > out.log 2>&1 &
                        '
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
            echo '❌ Deployment failed. Check logs above.'
        }
    }
}

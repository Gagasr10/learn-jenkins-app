pipeline {
    agent any

    stages {
        stage('Build') {
            agent {
                docker {
                    image 'node:18-alpine'
                    reuseNode true
                }
            }
            steps {
                sh '''
                    ls -la
                    node --version
                    npm --version
                    npm ci
                    npm run build
                    ls -la
                '''
            }
        }

        stage('Test') {
            agent {
                docker {
                    image 'node:18-alpine'
                    reuseNode true
                }
            }
            steps {
                sh '''
                    test -f build/index.html
                    npm test
                '''
            }
        }
        
        stage('E2E') {
            agent {
                docker {
                    image 'node:18-bullseye'  // PROMENJENO: Bullseye umesto Playwright
                    reuseNode true
                }
            }
            steps {
                sh '''
                    # Instaliraj Playwright
                    npm install @playwright/test
                    npx playwright install --with-deps chromium
                    
                    npm install serve
                    node_modules/.bin/serve -s build &
                    sleep 10
                    npx playwright test
                '''
            }
        }
    }

    post {
        always {
            junit 'jest-results/junit.xml'  // PROMENJENO: jest-results (sa 's')
            echo 'Pipeline completed!'
        }
    }
}
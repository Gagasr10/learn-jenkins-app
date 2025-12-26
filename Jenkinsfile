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
                    CI=true npm test
                '''
            }
        }

        stage('E2E') {
            agent {
                docker {
                    // Verzija usklađena sa @playwright/test ^1.39.0
                    image 'mcr.microsoft.com/playwright:v1.39.0-jammy'
                    reuseNode true
                }
            }
            steps {
                sh '''
                    node --version
                    npm --version

                    # Uveri se da dependencies postoje (i da playwright test runner postoji)
                    npm ci

                    # Startuj built app statički (bez global install-a)
                    npx serve -s build -l 3000 &
                    sleep 3

                    # Pokreni E2E (baseURL u playwright.config.js treba da gađa http://localhost:3000)
                    npx playwright test --reporter=html
                '''
            }
            post {
                always {
                    archiveArtifacts artifacts: 'playwright-report/**', allowEmptyArchive: true
                    archiveArtifacts artifacts: 'test-results/**', allowEmptyArchive: true
                }
            }
        }
    }

    post {
        always {
            junit 'jest-results/junit.xml'
            echo 'Pipeline completed!'
        }
    }
}

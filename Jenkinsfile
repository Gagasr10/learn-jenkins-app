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
            image 'mcr.microsoft.com/playwright:v1.39.0-jammy'
            reuseNode true
        }
    }
    steps {
        sh '''
            node --version
            npm --version

            npm ci

            # Start statičkog servera (bind na sve interfejse) + log u fajl
            npx serve -s build -l tcp://0.0.0.0:3000 > serve.log 2>&1 &
            
            # Čekaj da server stvarno bude dostupan (umesto sleep)
            npx --yes wait-on http://127.0.0.1:3000

            # Pokreni E2E
            npx playwright test --reporter=html
        '''
    }
    post {
        always {
            archiveArtifacts artifacts: 'serve.log', allowEmptyArchive: true
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

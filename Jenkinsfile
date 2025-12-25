pipeline {
    agent any

    stages {
        stage('Build') {
            steps {
                sh'''
                    docker run --rm -v $(pwd):/app -w /app node:18-alpine \
                    sh -c "ls -la &&
                           node --version &&
                           npm --version &&
                           npm ci &&
                           npm run build &&
                           ls -la"
                '''
            }
        }

        stage ('Test') {
            steps{
                echo 'Test stage'
                sh 'test -f build/index.html'
            }
        }
    }
}
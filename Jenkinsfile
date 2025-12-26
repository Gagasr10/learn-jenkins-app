pipeline {
  agent any

  stages {

    stage('Install') {
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
        '''
        stash name: 'deps', includes: 'node_modules/**,package.json,package-lock.json'
      }
    }

    stage('Build') {
      agent {
        docker {
          image 'node:18-alpine'
          reuseNode true
        }
      }
      steps {
        unstash 'deps'
        sh '''
          npm run build
          test -f build/index.html
          ls -la
        '''
        // build + e2e test fajlovi + playwright config
        stash name: 'app', includes: 'build/**,e2e/**,playwright.config.js'
      }
    }

    stage('Run Tests') {
      parallel {

        stage('Unit tests') {
          agent {
            docker {
              image 'node:18-alpine'
              reuseNode true
            }
          }
          steps {
            unstash 'deps'
            unstash 'app'
            sh '''
              CI=true npm test -- --watchAll=false
            '''
          }
          post {
            always {
              junit 'jest-results/junit.xml'
            }
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
            unstash 'deps'
            unstash 'app'
            sh '''
              # Start statičkog servera + log
              npx serve -s build -l tcp://0.0.0.0:3000 > serve.log 2>&1 &

              # čekaj da server bude ready
              npx --yes wait-on http://127.0.0.1:3000

              # E2E
              npx playwright test --reporter=html
            '''
          }
          post {
            always {
              archiveArtifacts artifacts: 'serve.log', allowEmptyArchive: true
              archiveArtifacts artifacts: 'playwright-report/**', allowEmptyArchive: true
              archiveArtifacts artifacts: 'test-results/**', allowEmptyArchive: true
              publishHTML([
                allowMissing: true,
                alwaysLinkToLastBuild: true,
                keepAll: true,
                reportDir: 'playwright-report',
                reportFiles: 'index.html',
                reportName: 'Playwright HTML Report'
              ])
            }
          }
        }

      }
    }
  }

  post {
    always {
      echo 'Pipeline completed!'
    }
  }
}

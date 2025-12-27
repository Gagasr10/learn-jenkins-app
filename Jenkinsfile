pipeline {
  agent any

  environment {
    NETLIFY_SITE_ID    = '48050a32-ad69-42cc-9c19-dd33ee11812b'
    NETLIFY_AUTH_TOKEN = credentials('netlify-token')
    BASE_URL           = 'https://astonishing-crostata-78f01e.netlify.app'
  }

  stages {

    /* ================= BUILD ================= */
    stage('Build') {
      agent {
        docker {
          image 'node:18-alpine'
          reuseNode true
        }
      }
      steps {
        sh '''
          node --version
          npm --version
          npm ci
          npm run build
          test -f build/index.html
        '''
        stash name: 'deps', includes: 'node_modules/**,package.json,package-lock.json'
        stash name: 'app',  includes: 'build/**,e2e/**,playwright.config.js'
      }
    }

    /* ================= TESTS ================= */
    stage('Tests') {
      parallel {

        /* ---------- UNIT TESTS ---------- */
        stage('Unit tests') {
          agent {
            docker {
              image 'node:18-alpine'
              reuseNode true
            }
          }
          steps {
            unstash 'deps'
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

        /* ---------- LOCAL E2E ---------- */
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
              npx serve -s build -l tcp://0.0.0.0:3000 > serve.log 2>&1 &
              npx --yes wait-on http://127.0.0.1:3000
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
                reportName: 'Playwright Local'
              ])
            }
          }
        }
      }
    }

    /* ================= DEPLOY ================= */
    stage('Deploy staging') {
  agent {
    docker {
      image 'node:18-alpine'
      reuseNode true
    }
  }
  steps {
    unstash 'app'
    sh '''
      npx netlify-cli --version
      echo "Deploying to staging. Site ID: $NETLIFY_SITE_ID"
      npx netlify-cli status

      # KLJUČ: spreči Netlify da pokušava build na svom serveru (bash ENOENT)
      npx netlify-cli deploy --dir=build --no-build
    '''
  }
}

    
    stage('Deploy prod') {
      agent {
        docker {
          image 'node:18-alpine'
          reuseNode true
        }
      }
      steps {
        unstash 'app'
        sh '''
          npx netlify-cli --version
          echo "Deploying to production. Site ID: $NETLIFY_SITE_ID"
          npx netlify-cli status
          npx netlify-cli deploy --dir=build --prod --no-build
        '''
      }
    }

    /* ================= PROD E2E ================= */
    stage('Prod E2E') {
      agent {
        docker {
          image 'mcr.microsoft.com/playwright:v1.39.0-jammy'
          reuseNode true
        }
      }
      steps {
        unstash 'deps'
        sh '''
          BASE_URL="$BASE_URL" npx playwright test --reporter=html
        '''
      }
      post {
        always {
          archiveArtifacts artifacts: 'playwright-report/**', allowEmptyArchive: true
          publishHTML([
            allowMissing: true,
            alwaysLinkToLastBuild: true,
            keepAll: true,
            reportDir: 'playwright-report',
            reportFiles: 'index.html',
            reportName: 'Playwright Prod'
          ])
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

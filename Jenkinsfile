pipeline {
  agent any

  environment {
    NETLIFY_SITE_ID     = '48050a32-ad69-42cc-9c19-dd33ee11812b'
    NETLIFY_AUTH_TOKEN  = credentials('netlify-token')
    PROD_URL            = 'https://astonishing-crostata-78f01e.netlify.app'
  }

  stages {

    stage('Install') {
      agent { docker { image 'node:18-alpine'; reuseNode true } }
      steps {
        sh '''
          node --version
          npm --version
          npm ci
        '''
        stash name: 'deps', includes: 'node_modules/**,package.json,package-lock.json'
      }
    }

    stage('Build') {
      agent { docker { image 'node:18-alpine'; reuseNode true } }
      steps {
        unstash 'deps'
        sh '''
          npm run build
          test -f build/index.html
        '''
        stash name: 'app', includes: 'build/**,e2e/**,playwright.config.js'
      }
    }

    stage('Tests') {
      parallel {

        stage('Unit tests') {
          agent { docker { image 'node:18-alpine'; reuseNode true } }
          steps {
            unstash 'deps'
            // build nije potreban za unit, ali nije problem ako ga dodaš; ostavljam čisto
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

        stage('Local E2E') {
          agent { docker { image 'mcr.microsoft.com/playwright:v1.39.0-jammy'; reuseNode true } }
          steps {
            unstash 'deps'
            unstash 'app'
            sh '''
              # startuj server (bez npm install serve)
              npx serve -s build -l tcp://0.0.0.0:3000 > serve.log 2>&1 &

              # sačekaj ready
              npx --yes wait-on http://127.0.0.1:3000

              # E2E
              npx playwright test --reporter=html
            '''
          }
          post {
            always {
              archiveArtifacts artifacts: 'serve.log', allowEmptyArchive: true
              publishHTML([
                allowMissing: true,
                alwaysLinkToLastBuild: true,
                keepAll: true,
                reportDir: 'playwright-report',
                reportFiles: 'index.html',
                reportName: 'Playwright Local'
              ])
              archiveArtifacts artifacts: 'playwright-report/**', allowEmptyArchive: true
              archiveArtifacts artifacts: 'test-results/**', allowEmptyArchive: true
            }
          }
        }
      }
    }

    stage('Deploy') {
      agent { docker { image 'node:18-alpine'; reuseNode true } }
      steps {
        unstash 'app'
        sh '''
          # netlify-cli bez instalacije u node_modules
          npx netlify-cli --version

          echo "Deploying to production. Site ID: $NETLIFY_SITE_ID"
          npx netlify-cli status

          # KLJUČ: --no-build (da Netlify ne pokreće "npm run build" na serveru)
          npx netlify-cli deploy --dir=build --prod --no-build
        '''
      }
    }

    stage('Prod E2E') {
      agent { docker { image 'mcr.microsoft.com/playwright:v1.39.0-jammy'; reuseNode true } }
      steps {
        unstash 'deps'
        sh '''
          # Pokreni prod testove ka Netlify URL-u
          BASE_URL="$PROD_URL" npx playwright test --reporter=html
        '''
      }
      post {
        always {
          publishHTML([
            allowMissing: true,
            alwaysLinkToLastBuild: true,
            keepAll: true,
            reportDir: 'playwright-report',
            reportFiles: 'index.html',
            reportName: 'Playwright Prod'
          ])
          archiveArtifacts artifacts: 'playwright-report/**', allowEmptyArchive: true
          archiveArtifacts artifacts: 'test-results/**', allowEmptyArchive: true
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

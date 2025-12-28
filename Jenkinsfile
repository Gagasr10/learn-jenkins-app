pipeline {
    agent any

    environment {
        NETLIFY_SITE_ID     = '48050a32-ad69-42cc-9c19-dd33ee11812b'
        NETLIFY_AUTH_TOKEN  = credentials('netlify-token')
    }

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

        stage('Tests') {
            parallel {
                stage('Unit tests') {
                    agent {
                        docker {
                            image 'node:18-alpine'
                            reuseNode true
                        }
                    }

                    steps {
                        sh '''
                            #test -f build/index.html
                            npm test
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
                        sh '''
                            npm install serve
                            node_modules/.bin/serve -s build &
                            sleep 10
                            npx playwright test --reporter=html
                        '''
                    }

                    post {
                        always {
                            publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, keepAll: false, reportDir: 'playwright-report', reportFiles: 'index.html', reportName: 'Local E2E', reportTitles: '', useWrapperFileDirectly: true])
                        }
                    }
                }
            }
        }

        stage('Deploy staging + Staging E2E') {
            agent {
                docker {
                    image 'mcr.microsoft.com/playwright:v1.39.0-jammy'
                    reuseNode true
                }
            }

            steps {
                sh '''
                    # Netlify CLI latest traži Node 20+, a Playwright image ti je Node 18.x
                    # Zato pinujemo verziju koja radi na Node 18.
                    npm install netlify-cli@21.6.0 node-jq

                    node_modules/.bin/netlify --version
                    echo "Deploying to staging. Site ID: $NETLIFY_SITE_ID"

                    # status bez --site (u nekim verzijama puca)
                    node_modules/.bin/netlify status || true

                    # deploy (upload build folder, bez build-a na Netlify)
                    node_modules/.bin/netlify deploy --dir=build --json --no-build \
                      --site "$NETLIFY_SITE_ID" \
                      --auth "$NETLIFY_AUTH_TOKEN" > deploy-output.json

                    STAGING_URL=$(node_modules/.bin/node-jq -r '.deploy_url' deploy-output.json)
                    echo "Staging URL: $STAGING_URL"

                    # Pokreni Playwright testove protiv staging URL-a
                    CI_ENVIRONMENT_URL="$STAGING_URL" npx playwright test --reporter=html
                '''
            }

            post {
                always {
                    publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, keepAll: false, reportDir: 'playwright-report', reportFiles: 'index.html', reportName: 'Staging E2E', reportTitles: '', useWrapperFileDirectly: true])
                }
            }
        }

        stage('Approval') {
            steps {
                timeout(time: 15, unit: 'MINUTES') {
                    input message: 'Do you wish to deploy to production?', ok: 'Yes, I am sure!'
                }
            }
        }

        stage('Deploy prod + Prod E2E') {
            agent {
                docker {
                    image 'mcr.microsoft.com/playwright:v1.39.0-jammy'
                    reuseNode true
                }
            }

            steps {
                sh '''
                    # Pin Netlify CLI zbog Node 18 u Playwright image-u
                    npm install netlify-cli@21.6.0

                    node_modules/.bin/netlify --version
                    echo "Deploying to production. Site ID: $NETLIFY_SITE_ID"

                    node_modules/.bin/netlify status || true

                    # prod deploy (bez build-a)
                    node_modules/.bin/netlify deploy --dir=build --prod --no-build \
                      --site "$NETLIFY_SITE_ID" \
                      --auth "$NETLIFY_AUTH_TOKEN"

                    # Prod URL ti je stalni Netlify URL projekta
                    CI_ENVIRONMENT_URL="https://astonishing-crostata-78f01e.netlify.app" npx playwright test --reporter=html
                '''
            }

            post {
                always {
                    publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, keepAll: false, reportDir: 'playwright-report', reportFiles: 'index.html', reportName: 'Prod E2E', reportTitles: '', useWrapperFileDirectly: true])
                }
            }
        }
    }
}

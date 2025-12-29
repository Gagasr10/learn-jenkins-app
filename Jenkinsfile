pipeline {
    agent any

    environment {
        NETLIFY_SITE_ID     = '48050a32-ad69-42cc-9c19-dd33ee11812b'
        NETLIFY_AUTH_TOKEN  = credentials('netlify-token')
        REACT_APP_VERSION   = "1.0.${BUILD_ID}"
    }

    options { timestamps() }

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
                    set -e
                    echo "REACT_APP_VERSION=$REACT_APP_VERSION"
                    node --version
                    npm --version

                    npm ci
                    export REACT_APP_VERSION="$REACT_APP_VERSION"
                    npm run build

                    test -f build/index.html
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
                            set -e
                            npm ci
                            npm test
                        '''
                    }
                    post {
                        always {
                            junit 'jest-results/junit.xml'
                        }
                    }
                }

                stage('E2E (Local build)') {
                    agent {
                        docker {
                            image 'mcr.microsoft.com/playwright:v1.39.0-jammy'
                            reuseNode true
                        }
                    }
                    steps {
                        sh '''
                            set -e

                            # očisti stare rezultate da publishHTML ne poludi
                            rm -rf playwright-report test-results || true

                            # instaliraj dependencies (mora zbog @playwright/test)
                            npm ci

                            # (opciono) osiguraj browser deps ako ikad fali
                            # npx playwright install --with-deps

                            # startuj statički server
                            npx serve -s build -l 3000 >/tmp/serve.log 2>&1 &
                            SERVE_PID=$!

                            # čekaj da port proradi
                            for i in $(seq 1 20); do
                              if curl -sSf http://127.0.0.1:3000 >/dev/null; then
                                echo "Server is up."
                                break
                              fi
                              echo "Waiting for server..."
                              sleep 1
                            done

                            # pokreni testove
                            npx playwright test --reporter=html

                            # ugasi server
                            kill $SERVE_PID || true
                        '''
                    }
                    post {
                        always {
                            publishHTML([
                                allowMissing: false,
                                alwaysLinkToLastBuild: false,
                                keepAll: false,
                                reportDir: 'playwright-report',
                                reportFiles: 'index.html',
                                reportName: 'Local E2E',
                                reportTitles: '',
                                useWrapperFileDirectly: true
                            ])
                        }
                    }
                }
            }
        }

        stage('Deploy staging') {
            agent {
                docker {
                    image 'node:18-alpine'
                    reuseNode true
                }
            }
            steps {
                sh '''
                    set -e
                    npm ci
                    npm i -D netlify-cli node-jq

                    echo "Deploying to STAGING. Site ID: $NETLIFY_SITE_ID"
                    npx netlify status

                    npx netlify deploy --dir=build --json > deploy-output.json
                    export CI_ENVIRONMENT_URL=$(npx node-jq -r '.deploy_url' deploy-output.json)

                    echo "Staging URL: $CI_ENVIRONMENT_URL"

                    # (ako želiš staging E2E ovde, onda prebaci u playwright image + npm ci)
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
                sh '''
                    set -e
                    npm ci
                    npm i -D netlify-cli

                    echo "Deploying to PROD. Site ID: $NETLIFY_SITE_ID"
                    npx netlify status
                    npx netlify deploy --dir=build --prod
                '''
            }
        }
    }
}

pipeline {
    agent any

    environment {
        NETLIFY_SITE_ID     = '48050a32-ad69-42cc-9c19-dd33ee11812b'
        NETLIFY_AUTH_TOKEN  = credentials('netlify-token')

        // Bitno: verziju formiramo preko Jenkins env var-a (stabilno)
        REACT_APP_VERSION   = "1.0.${env.BUILD_ID}"
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
                    set -e
                    ls -la
                    node --version
                    npm --version

                    # React čita env tokom build-a
                    export REACT_APP_VERSION="$REACT_APP_VERSION"

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
                            set -e
                            export REACT_APP_VERSION="$REACT_APP_VERSION"
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
                            set -e
                            export REACT_APP_VERSION="$REACT_APP_VERSION"

                            npm install serve
                            node_modules/.bin/serve -s build &
                            sleep 10

                            npx playwright test --reporter=html
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
                    image 'mcr.microsoft.com/playwright:v1.39.0-jammy'
                    reuseNode true
                }
            }

            steps {
                sh '''
                    set -e
                    export REACT_APP_VERSION="$REACT_APP_VERSION"

                    # Pinujemo verziju netlify-cli koja radi na Node 18 (bez engine warning-a / potencijalnog fail-a)
                    npm install netlify-cli@21.6.0 node-jq
                    node_modules/.bin/netlify --version

                    echo "Deploying to staging. Site ID: $NETLIFY_SITE_ID"

                    # Direktno deploy (status izbacen jer ti puca na --site)
                    node_modules/.bin/netlify deploy \
                      --site "$NETLIFY_SITE_ID" \
                      --auth "$NETLIFY_AUTH_TOKEN" \
                      --dir=build \
                      --json > deploy-output.json

                    export CI_ENVIRONMENT_URL=$(node_modules/.bin/node-jq -r '.deploy_url' deploy-output.json)

                    echo "Staging URL: $CI_ENVIRONMENT_URL"

                    # E2E na staging
                    npx playwright test --reporter=html
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
                        reportName: 'Staging E2E',
                        reportTitles: '',
                        useWrapperFileDirectly: true
                    ])
                }
            }
        }

        stage('Deploy prod') {
            agent {
                docker {
                    image 'mcr.microsoft.com/playwright:v1.39.0-jammy'
                    reuseNode true
                }
            }

            environment {
                // Ostaje placeholder dok ne zalepis svoj production URL
                CI_ENVIRONMENT_URL = 'YOUR NETLIFY SITE URL'
            }

            steps {
                sh '''
                    set -e
                    export REACT_APP_VERSION="$REACT_APP_VERSION"

                    npm install netlify-cli@21.6.0
                    node_modules/.bin/netlify --version

                    echo "Deploying to production. Site ID: $NETLIFY_SITE_ID"

                    node_modules/.bin/netlify deploy \
                      --site "$NETLIFY_SITE_ID" \
                      --auth "$NETLIFY_AUTH_TOKEN" \
                      --dir=build \
                      --prod

                    # E2E na production (tek kad setujes CI_ENVIRONMENT_URL kako treba)
                    npx playwright test --reporter=html
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
                        reportName: 'Prod E2E',
                        reportTitles: '',
                        useWrapperFileDirectly: true
                    ])
                }
            }
        }
    }
}

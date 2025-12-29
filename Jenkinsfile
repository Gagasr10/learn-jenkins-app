pipeline {
    agent any

    environment {
        NETLIFY_SITE_ID     = '48050a32-ad69-42cc-9c19-dd33ee11812b'
        NETLIFY_AUTH_TOKEN  = credentials('netlify-token')
        REACT_APP_VERSION   = "1.0.${BUILD_ID}"
        // CI_ENVIRONMENT_URL se postavlja u deploy stage-ovima
    }

    options {
        timestamps()
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
                    echo "REACT_APP_VERSION=$REACT_APP_VERSION"
                    node --version
                    npm --version

                    npm ci
                    export REACT_APP_VERSION="$REACT_APP_VERSION"
                    npm run build

                    test -f build/index.html
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
                            # Playwright koristi webServer iz playwright.config.js
                            # i sam startuje: npx serve -s build -l 3000
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
                    image 'node:18-alpine'
                    reuseNode true
                }
            }

            steps {
                sh '''
                    set -e
                    node --version
                    npm --version

                    npm ci
                    npm i -D netlify-cli node-jq
                    npx netlify --version

                    echo "Deploying to STAGING. Site ID: $NETLIFY_SITE_ID"
                    npx netlify status

                    npx netlify deploy --dir=build --json > deploy-output.json
                    export CI_ENVIRONMENT_URL=$(npx node-jq -r '.deploy_url' deploy-output.json)

                    echo "Staging URL: $CI_ENVIRONMENT_URL"

                    # E2E protiv staging URL-a
                    CI_ENVIRONMENT_URL="$CI_ENVIRONMENT_URL" npx playwright test --reporter=html
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
                    image 'node:18-alpine'
                    reuseNode true
                }
            }

            steps {
                sh '''
                    set -e
                    node --version
                    npm --version

                    npm ci
                    npm i -D netlify-cli
                    npx netlify --version

                    echo "Deploying to PROD. Site ID: $NETLIFY_SITE_ID"
                    npx netlify status

                    npx netlify deploy --dir=build --prod

                    # Ako hoćeš da testiraš prod URL:
                    # export CI_ENVIRONMENT_URL="https://YOUR_SITE.netlify.app"
                    # CI_ENVIRONMENT_URL="$CI_ENVIRONMENT_URL" npx playwright test --reporter=html
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

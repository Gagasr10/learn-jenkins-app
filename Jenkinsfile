pipeline {
    agent any

    environment {
        NETLIFY_SITE_ID     = '48050a32-ad69-42cc-9c19-dd33ee11812b'
        NETLIFY_AUTH_TOKEN  = credentials('netlify-token')
        REACT_APP_VERSION   = "1.0.${BUILD_ID}"
    }

    options {
        timestamps()
    }

    stages {

        stage('Docker') {
            steps {
                sh '''
                    set -e
                    docker build -t my-playwright .
                '''
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
                sh '''
                    set -e
                    echo "REACT_APP_VERSION=$REACT_APP_VERSION"
                    export REACT_APP_VERSION="$REACT_APP_VERSION"

                    node --version
                    npm --version

                    npm ci
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
                            echo "REACT_APP_VERSION=$REACT_APP_VERSION"
                            export REACT_APP_VERSION="$REACT_APP_VERSION"

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
                            echo "REACT_APP_VERSION=$REACT_APP_VERSION"
                            export REACT_APP_VERSION="$REACT_APP_VERSION"

                            # očisti stare reporte da publishHTML ne puca
                            rm -rf playwright-report test-results || true

                            # instaliraj devDependencies (uključuje @playwright/test)
                            npm ci

                            # Playwright će podići server kroz webServer u playwright.config.js
                            npx playwright test --reporter=html
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
                    image 'my-playwright'
                    reuseNode true
                }
            }

            environment {
                CI_ENVIRONMENT_URL = 'STAGING_URL_TO_BE_SET'
            }

            steps {
                sh '''
                    set -e
                    echo "Deploying to staging. Site ID: $NETLIFY_SITE_ID"

                    # netlify i node-jq su u tvom my-playwright image-u (kako si planirao)
                    netlify --version

                    # deploy (preview)
                    netlify deploy --site "$NETLIFY_SITE_ID" --dir=build --json > deploy-output.json

                    # izvuci deploy_url i prosledi Playwright-u
                    export CI_ENVIRONMENT_URL="$(node-jq -r '.deploy_url' deploy-output.json)"
                    echo "STAGING URL: $CI_ENVIRONMENT_URL"

                    rm -rf playwright-report test-results || true
                    npx playwright test --reporter=html
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
                    image 'my-playwright'
                    reuseNode true
                }
            }

            environment {
                CI_ENVIRONMENT_URL = 'YOUR_NETLIFY_SITE_URL'
            }

            steps {
                sh '''
                    set -e
                    echo "Deploying to production. Site ID: $NETLIFY_SITE_ID"

                    netlify --version

                    # production deploy
                    netlify deploy --site "$NETLIFY_SITE_ID" --dir=build --prod

                    rm -rf playwright-report test-results || true
                    npx playwright test --reporter=html
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
                        reportName: 'Prod E2E',
                        reportTitles: '',
                        useWrapperFileDirectly: true
                    ])
                }
            }
        }
    }
}

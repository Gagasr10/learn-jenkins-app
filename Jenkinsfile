pipeline {
    agent any

    environment {
        NETLIFY_SITE_ID     = '48050a32-ad69-42cc-9c19-dd33ee11812b'
        NETLIFY_AUTH_TOKEN  = credentials('netlify-token')
        REACT_APP_VERSION   = "1.0.${BUILD_ID}"
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

                    echo "REACT_APP_VERSION (build): $REACT_APP_VERSION"

                    npm ci
                    export REACT_APP_VERSION="$REACT_APP_VERSION"
                    npm run build

                    ls -la
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
                            npm test -- --watchAll=false
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
                            echo "REACT_APP_VERSION (e2e expected): $REACT_APP_VERSION"

                            npm install serve
                            node_modules/.bin/serve -s build -l 3000 &
                            sleep 10

                            export REACT_APP_VERSION="$REACT_APP_VERSION"
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

                    # Pin netlify-cli to a Node18-friendly version
                    npm install netlify-cli@21.6.0 node-jq

                    node_modules/.bin/netlify --version
                    echo "Deploying to staging. Site ID: $NETLIFY_SITE_ID"

                    # Avoid "netlify link" dependency: pass site/auth explicitly
                    node_modules/.bin/netlify deploy \
                      --dir=build \
                      --site="$NETLIFY_SITE_ID" \
                      --auth="$NETLIFY_AUTH_TOKEN" \
                      --json > deploy-output.json

                    echo "Staging URL:"
                    node_modules/.bin/node-jq -r '.deploy_url' deploy-output.json
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

                    npm install netlify-cli@21.6.0
                    node_modules/.bin/netlify --version

                    echo "Deploying to production. Site ID: $NETLIFY_SITE_ID"

                    node_modules/.bin/netlify deploy \
                      --dir=build \
                      --prod \
                      --site="$NETLIFY_SITE_ID" \
                      --auth="$NETLIFY_AUTH_TOKEN"
                '''
            }
        }
    }
}

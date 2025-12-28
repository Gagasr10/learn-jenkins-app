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

                    echo "REACT_APP_VERSION=$REACT_APP_VERSION"

                    # IMPORTANT: CRA čita REACT_APP_* tokom build-a
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

                            # koristimo npx da ne prljamo node_modules sa "serve"
                            npx serve -s build -l 3000 &
                            SERVE_PID=$!
                            sleep 5

                            npx playwright test --reporter=html

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
                    image 'mcr.microsoft.com/playwright:v1.39.0-jammy'
                    reuseNode true
                }
            }

            environment {
                CI_ENVIRONMENT_URL = 'STAGING_URL_TO_BE_SET'
            }

            steps {
                sh '''
                    set -e
                    export REACT_APP_VERSION="$REACT_APP_VERSION"

                    # Pinujemo netlify-cli na verziju koja radi na Node 18
                    # (novije verzije traže Node >= 20.x)
                    npx netlify-cli@18.0.1 --version

                    echo "Deploying to staging. Site ID: $NETLIFY_SITE_ID"

                    # Deploy (staging preview) + json output
                    npx netlify-cli@18.0.1 deploy \
                      --dir=build \
                      --json \
                      --site "$NETLIFY_SITE_ID" \
                      --auth "$NETLIFY_AUTH_TOKEN" \
                      > deploy-output.json

                    echo "Deploy output:"
                    cat deploy-output.json

                    # Izvuci deploy_url bez node-jq
                    node -e "const j=require('./deploy-output.json'); console.log('CI_ENVIRONMENT_URL=' + (j.deploy_url || ''));"
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
                CI_ENVIRONMENT_URL = 'YOUR_NETLIFY_SITE_URL'
            }

            steps {
                sh '''
                    set -e
                    export REACT_APP_VERSION="$REACT_APP_VERSION"

                    npx netlify-cli@18.0.1 --version

                    echo "Deploying to production. Site ID: $NETLIFY_SITE_ID"

                    npx netlify-cli@18.0.1 deploy \
                      --dir=build \
                      --prod \
                      --site "$NETLIFY_SITE_ID" \
                      --auth "$NETLIFY_AUTH_TOKEN"
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

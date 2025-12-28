pipeline {
    agent any

    environment {
        // TODO: zameni svojim Netlify Site ID (iz Netlify dashboard-a)
        NETLIFY_SITE_ID = 'YOUR_NETLIFY_SITE_ID'

        // Jenkins credential ID: netlify-token (Secret text)
        NETLIFY_AUTH_TOKEN = credentials('netlify-token')

        // Verzija aplikacije koju prikazuje App.js (REACT_APP_ prefiks je bitan za CRA)
        REACT_APP_VERSION = "1.0.${BUILD_ID}"
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
                    npm run build

                    ls -la
                    test -d build
                '''

                // Sačuvaj build da ga koristiš u sledećim stage-ovima (docker konteneri su odvojeni)
                stash name: 'build', includes: 'build/**'
                stash name: 'repo_files', includes: 'package.json,package-lock.json,playwright.config.js,e2e/**,src/**,public/**'
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
                            node --version
                            npm --version

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

                stage('E2E') {
                    agent {
                        docker {
                            image 'mcr.microsoft.com/playwright:v1.39.0-jammy'
                            reuseNode true
                        }
                    }
                    steps {
                        // Vrati build folder i fajlove koji trebaju testovima
                        unstash 'build'
                        unstash 'repo_files'

                        sh '''
                            set -e
                            echo "REACT_APP_VERSION=$REACT_APP_VERSION"
                            node --version
                            npm --version

                            # instaliraj dependencies za e2e (playwright + ostalo)
                            npm ci

                            echo "Starting static server on :3000 ..."
                            # start server in background
                            npx serve -s build -l 3000 > serve-ci.log 2>&1 &
                            SERVE_PID=$!
                            echo "SERVE_PID=$SERVE_PID"

                            # wait until server responds (max ~30s)
                            node - <<'NODE'
                            const http = require('http');
                            const url = 'http://127.0.0.1:3000/';
                            let attempts = 0;
                            const max = 30;

                            function check() {
                              attempts++;
                              const req = http.get(url, (res) => {
                                if (res.statusCode && res.statusCode >= 200 && res.statusCode < 500) {
                                  console.log('Server is up:', res.statusCode);
                                  process.exit(0);
                                }
                                res.resume();
                                retry();
                              });
                              req.on('error', retry);
                            }

                            function retry() {
                              if (attempts >= max) {
                                console.error('Server did not become ready in time.');
                                process.exit(1);
                              }
                              setTimeout(check, 1000);
                            }

                            check();
NODE

                            echo "Running Playwright..."
                            npx playwright test --reporter=html

                            echo "Stopping server..."
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
                unstash 'build'

                sh '''
                    set -e
                    echo "Deploying to staging. Site ID: $NETLIFY_SITE_ID"
                    node --version
                    npm --version

                    npm install -g netlify-cli
                    netlify --version

                    # Netlify status (može fail ako nije linkovano lokalno - nije fatalno)
                    netlify status || true

                    # Deploy staging i izvuci deploy URL
                    netlify deploy --dir=build --json > deploy-output.json

                    node - <<'NODE'
                    const fs = require('fs');
                    const data = JSON.parse(fs.readFileSync('deploy-output.json','utf8'));
                    console.log('STAGING_DEPLOY_URL=' + data.deploy_url);
NODE

                    # Ako želiš da koristiš staging URL u Playwright testu,
                    # treba da podržiš u playwright.config.js baseURL iz env var.
                    # npx playwright test --reporter=html
                '''
            }

            post {
                always {
                    archiveArtifacts artifacts: 'deploy-output.json', fingerprint: true
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

            environment {
                // TODO: zameni svojim PRODUCTION Netlify URL (npr. https://tvoj-sajt.netlify.app)
                CI_ENVIRONMENT_URL = 'YOUR_NETLIFY_SITE_URL'
            }

            steps {
                unstash 'build'

                sh '''
                    set -e
                    echo "Deploying to production. Site ID: $NETLIFY_SITE_ID"
                    node --version
                    npm --version

                    npm install -g netlify-cli
                    netlify --version

                    netlify status || true
                    netlify deploy --dir=build --prod

                    # Ako želiš Playwright protiv PRODUCTION:
                    # npx playwright test --reporter=html
                '''
            }
        }
    }
}

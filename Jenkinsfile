pipeline {
    agent any ##{
        label 'rhel8'  // Specify RHEL 8 agent label
    }

    environment {
        NETLIFY_SITE_ID = '53b63b0f-3194-4bd0-96af-ce233da908d1'
        NETLIFY_AUTH_TOKEN = credentials('netlify-token')
        NODE_VERSION = '18'
        PATH = "/usr/local/node/bin:$PATH"
    }

    stages {

        stage('Setup Node.js') {
            steps {
                sh '''
                    # Install Node.js 18 if not present
                    if ! command -v node &> /dev/null || [[ $(node --version | cut -d'v' -f2 | cut -d'.' -f1) -lt 18 ]]; then
                        echo "Installing Node.js 18..."
                        curl -fsSL https://rpm.nodesource.com/setup_18.x | sudo bash -
                        sudo dnf install -y nodejs
                    fi
                    
                    # Verify installation
                    node --version
                    npm --version
                '''
            }
        }

        stage('Build') {
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
                    steps {
                        sh '''
                            test -f build/index.html
                            npm test
                        '''
                    }
                    post {
                        always {
                            junit 'test-results/junit.xml'
                        }
                    }
                }

                stage('E2E') {
                    steps {
                        sh '''
                            # Install Playwright dependencies for RHEL 8
                            sudo dnf install -y \
                                libX11 libX11-xcb libxcb libxkbcommon \
                                libXcomposite libXcursor libXdamage libXext \
                                libXfixes libXi libXrandr libXrender libXss \
                                libXtst cups-libs libdrm libgtk-3 libxss1 \
                                libasound2 libatk-bridge2.0-0 libdrm2 \
                                libxkbcommon0 libatspi2.0-0 libxcomposite1 \
                                libxdamage1 libxrandr2 libgbm1 libxss1 \
                                libasound2 || true
                            
                            # Install Playwright
                            npm install @playwright/test
                            npx playwright install
                            npx playwright install-deps
                            
                            # Start server and run tests
                            npm install serve
                            node_modules/.bin/serve -s build &
                            SERVER_PID=$!
                            sleep 10
                            
                            # Run Playwright tests
                            npx playwright test --reporter=html
                            
                            # Clean up
                            kill $SERVER_PID || true
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
                                reportName: 'Playwright HTML Report', 
                                reportTitles: '', 
                                useWrapperFileDirectly: true
                            ])
                        }
                    }
                }
            }
        }

        stage('Deploy') {
            steps {
                sh '''
                    npm install netlify-cli
                    node_modules/.bin/netlify --version
                    echo "Deploying to production. Site ID: $NETLIFY_SITE_ID"
                    node_modules/.bin/netlify status
                    node_modules/.bin/netlify deploy --dir=build --prod
                '''
            }
        }
    }

    post {
        always {
            cleanWs()
        }
        failure {
            emailext (
                subject: "Pipeline Failed: ${env.JOB_NAME} - ${env.BUILD_NUMBER}",
                body: "Build failed. Check console output at ${env.BUILD_URL}",
                to: "${env.CHANGE_AUTHOR_EMAIL}"
            )
        }
    }
}

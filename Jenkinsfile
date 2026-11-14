pipeline {
    agent any

    environment {
        NETLIFY_SITE_ID = '268f2e11-b85e-481f-b72b-2e6304d0e875'
        NETLIFY_AUTH_TOKEN = credentials('netlify-token')
    }

    stages {

        // 1. BUILD
        stage('Build') {
            steps {
                bat '''
                    docker run --rm -v "%WORKSPACE%:/src" -v npm-cache:/root/.npm node:18-alpine sh -c "mkdir /app && cd /src && tar cf - --exclude=node_modules --exclude=build --exclude=.git . | tar xf - -C /app && cd /app && chmod -R u+w . && npm ci && npm run build && rm -rf /src/build && cp -r /app/build /src/build"
                '''
            }
        }

        // 2. TEST
        stage('Test') {
            steps {
                bat '''
                    docker run --rm -v "%WORKSPACE%:/src" -v npm-cache:/root/.npm node:18-alpine sh -c "test -f /src/build/index.html && echo build/index.html exists && mkdir /app && cd /src && tar cf - --exclude=node_modules --exclude=build --exclude=.git . | tar xf - -C /app && cd /app && chmod -R u+w . && npm ci && export CI=true JEST_JUNIT_OUTPUT_DIR=/app/test-results JEST_JUNIT_OUTPUT_NAME=junit.xml && npm test -- --reporters=default --reporters=jest-junit; rc=$?; mkdir -p /src/test-results; cp -r /app/test-results/. /src/test-results/; exit $rc"
                '''
            }
        }

        // 3. END-TO-END TESTING
        stage('E2E') {
            steps {
                retry(3) {
                    bat '''
                        docker run --rm -v "%WORKSPACE%:/src" -v npm-cache:/root/.npm mcr.microsoft.com/playwright:v1.39.0-jammy sh -c "mkdir /app && cd /src && tar cf - --exclude=node_modules --exclude=build --exclude=.git --exclude=playwright-report . | tar xf - -C /app && cd /app && chmod -R u+w . && cp -r /src/build /app/build && npm ci && npm install serve && (node_modules/.bin/serve -s build > /dev/null 2>&1 &) && sleep 10 && npx playwright test --reporter=html; rc=$?; rm -rf /src/playwright-report; cp -r /app/playwright-report /src/playwright-report; exit $rc"
                    '''
                }
            }

            post {
                always {
                    publishHTML(target: [
                        allowMissing: true,
                        alwaysLinkToLastBuild: false,
                        keepAll: false,
                        reportDir: 'playwright-report',
                        reportFiles: 'index.html',
                        reportName: 'Playwright Report'
                    ])
                }
            }
        }

        // 4. DEPLOY TO NETLIFY
        stage('Deploy') {
            steps {
                retry(3) {
                    bat '''
                        docker run --rm ^
                            -e NETLIFY_SITE_ID ^
                            -e NETLIFY_AUTH_TOKEN ^
                            -v "%WORKSPACE%:/src" ^
                            node:18-alpine sh -c "mkdir /tmp/deploy && cd /tmp/deploy && npm install netlify-cli --ignore-scripts && cp -r /src/build ./build && node_modules/.bin/netlify deploy --dir=build --prod --no-build --site=$NETLIFY_SITE_ID --auth=$NETLIFY_AUTH_TOKEN"
                    '''
                }
            }
        }
    }

    // POST-BUILD ACTIONS
    post {
        always {
            junit allowEmptyResults: true,
                  testResults: 'test-results/junit.xml'
        }
    }
}
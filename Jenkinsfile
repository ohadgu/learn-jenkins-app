pipeline {
    agent any

    environment {
        NETLIFY_SITE_ID = 'd7848e16-62a8-4c35-96af-581af18496b6'
        NETLIFY_AUTH_TOKEN = credentials('netlify-token') 
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
                    node --version
                    npm --version
                    npm ci
                    npm run build
                '''
            }
        }

        stage('Tests') {
            parallel {
                stage('Unit Test') {
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
                            publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, icon: '', keepAll: false, reportDir: 'playwright-report', reportFiles: 'index.html', reportName: 'Playwright Local', reportTitles: '', useWrapperFileDirectly: true])
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
                    npm install netlify-cli@20.1.1
                    npm install node-jq
                    node_modules/.bin/netlify --version
                    echo "Deploying to ** staging **. SITE_ID: $NETLIFY_SITE_ID"
                    npx netlify status
                    npx netlify deploy --dir=build --json > deploy-output.json
                    npx node-jq -r '.deploy_url' deploy-output.json
                '''
                script {
                    env.STAGING_URL = sh(
                        script: "npx node-jq -r '.deploy_url' deploy-output.json",
                        returnStdout: true
                    ).trim()
                }
            }
        }

        stage('Staging E2E') {
            agent {
                docker {
                    image 'mcr.microsoft.com/playwright:v1.39.0-jammy'
                    reuseNode true
                }
            }

            environment {
                CI_ENVIRONMENT_URL = "${env.STAGING_URL}"
            }

            steps {
                sh '''
                    npx playwright test --reporter=html
                '''
            }

            post {
                always {
                    publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, icon: '', keepAll: false, reportDir: 'playwright-report', reportFiles: 'index.html', reportName: 'Staging E2E', reportTitles: '', useWrapperFileDirectly: true])
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

        // stage('Deploy production') {
        //     agent {
        //         docker {
        //             image 'node:18-alpine'
        //             reuseNode true
        //         }
        //     }
        //     steps {
        //         sh '''
        //             npm install netlify-cli@20.1.1
        //             node_modules/.bin/netlify --version
        //             echo "Deploying to ** production **. SITE_ID: $NETLIFY_SITE_ID"
        //             npx netlify status
        //             npx netlify deploy --dir=build --prod
        //         '''
        //     }
        // }

        stage('Prod Deploy & E2E') {
            agent {
                docker {
                    image 'mcr.microsoft.com/playwright:v1.39.0-jammy'
                    reuseNode true
                }
            }

            environment {
                CI_ENVIRONMENT_URL = 'https://sweet-gnome-38f43d.netlify.app'
            }

            steps {
                sh '''
                    node --version
                    npm install netlify-cli@20.1.1
                    node_modules/.bin/netlify --version
                    echo "Deploying to ** production **. SITE_ID: $NETLIFY_SITE_ID"
                    npx netlify status
                    npx netlify deploy --dir=build --prod
                    npx playwright test --reporter=html
                '''
            }

            post {
                always {
                    publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, icon: '', keepAll: false, reportDir: 'playwright-report', reportFiles: 'index.html', reportName: 'Production E2E', reportTitles: '', useWrapperFileDirectly: true])
                }
            }
        } 
    }
}

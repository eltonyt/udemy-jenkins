#Notes

## Introduction

- Free and open source automation server - Build & Test
    - Compile Code
    - Run Tests
    - Build a new version of the app
    - Deploy the app
- Prerequisite
    - Docker Desktop
    - Jenkins Image
    - Installation - Need installation password - can be found within the container with Jenkins ran in
    - Administrator Registration
- Examples
    - Freestyle - script - Hello From Jenkins Script
        - **Freestyle jobs do not natively support the “pipeline as code” principle, making it difficult to track configuration changes**
    - Pipeline - pipeline script
        - to run shell script, we have to add `sh` at front within the pipeline script
            
            ```bash
            pipeline {
                agent any
            
                stages {
                    stage('Hello') {
                        steps {
                            echo 'Hello World'
                            sh 'echo "Hello from Jenkins"'
                        }
                    }
                }
            }
            ```
            
- Architecture
    - Jenkins Server (Controller)
        - Stores the results
        - Manages the pipelines and the associated jobs
    - Jenkins Agents
        - Executes the job
        - Report the results back to the Controller
- Jenkins Workspace
    - Likely a resource fold which stores all resources
    - To clean the workspace, we can add another process called “Post Actions”
    - Example
        
        ```bash
        pipeline {
            agent any
        
            stages {
                stage('Build') {
                    steps {
                        echo 'Building a new laptop ...'
                        // sh 'mkdir build'
                        sh 'touch build/computer.txt'
                        sh 'echo "mainboard" >> build/computer.txt'
                        sh 'cat build/computer.txt'
                    }
                }
            }
            
            post {
                always {
                    cleanWs()
                }
            }
        }
        
        ```
        
- Jenkins Artifact
    - Used to store the results from the last built - If workspace is cleaned every time, this is important
        
        ```bash
        pipeline {
            agent any
        
            stages {
                stage('Build') {
                    steps {
                        cleanWs()
                        echo 'Building a new laptop ...'
                        sh 'mkdir build'
                        sh 'touch build/computer.txt'
                        sh 'echo "mainboard" >> build/computer.txt'
                        sh 'cat build/computer.txt'
                    }
                }
            }
            
            post {
                success {
                    archiveArtifacts artifacts: 'build/**'   
                } 
            }
        }
        
        ```
        
- Combining multiple shell steps together
    
    ```bash
    pipeline {
        agent any
    
        stages {
            stage('Build') {
                steps {
                    cleanWs()
                    echo 'Building a new laptop ...'
                    **sh '''
                        mkdir -p build
                        touch build/computer.txt
                        echo "Mainboard" >> build/computer.txt
                        cat build/computer.txt
                        echo "Display" >> build/computer.txt
                        cat build/computer.txt
                        echo "Keyboard" >> build/computer.txt
                        cat build/computer.txt
                    '''**
                }
            }
        }
    
        post {
            success {
                archiveArtifacts artifacts: 'build/**'
            }
        }
    }
    ```
    
- Pipeline Stages
    - Test Stage
        
        ```bash
        pipeline {
            agent any
        
            stages {
                stage('Build') {
                    steps {
                        cleanWs()
                        echo 'Building a new laptop ...'
                        sh '''
                            mkdir -p build
                            touch build/computer.txt
                            echo "Mainboard" >> build/computer.txt
                            cat build/computer.txt
                            echo "Display" >> build/computer.txt
                            cat build/computer.txt
                            echo "Keyboard" >> build/computer.txt
                            cat build/computer.txt
                            rm build/computer.txt
                        '''
                    }
                }
                stage('Test') {
                    steps {
                        echo 'Testing the new laptop ...'
                        sh 'test -f build/computer.txt'
                    }
                }
            }
        
            post {
                success {
                    archiveArtifacts artifacts: 'build/**'
                }
            }
        }
        
        ```
        
- Defining environment variables
    - `environment {}`
        
        ```bash
        pipeline {
            agent any
        
            environment {
                BUILD_FILE_NAME = 'laptop.txt'
            }
        
            stages {
                stage('Build') {
                    steps {
                        cleanWs()
                        echo 'Building a new laptop ...'
                        sh '''
                            echo $BUILD_FILE_NAME
                            mkdir -p build
                            echo "Mainboard" >> build/$BUILD_FILE_NAME
                            cat build/$BUILD_FILE_NAME
                            echo "Display" >> build/$BUILD_FILE_NAME
                            cat build/$BUILD_FILE_NAME
                            echo "Keyboard" >> build/$BUILD_FILE_NAME
                            cat build/$BUILD_FILE_NAME
                        '''
                    }
                }
                stage('Test') {
                    steps {
                        echo 'Testing the new laptop ...'
                        sh '''
                            test -f build/$BUILD_FILE_NAME
                            grep "Mainboard" build/$BUILD_FILE_NAME
                            grep "Display" build/$BUILD_FILE_NAME
                            grep "Keyboard" build/$BUILD_FILE_NAME
                        '''
                    }
                }
            }
        
            post {
                success {
                    archiveArtifacts artifacts: 'build/**'
                }
            }
        }
        
        ```
        

## Implementing Continuous Integration (CI) with Jenkins

- Git (Track Code Change) → Build Automatically → Test Automatically → Deploy
- Jenkins build environment
    - Install all tools needed (like Node.js/npm) on the Jenkins agent
    - Use Docker with containers
        - need to enable docker docker pipeline plugins
        - Jenkins Dashboard → Manage Jenkins → Plugins → Available Plugins → Docker Pipeline
    - Example:
        
        ```bash
        pipeline {
            agent any
        
            stages {
                stage('w/o docker') {
                    steps {
                        sh 'echo "Without docker"'
                    }
                }
        
                stage('w/ docker') {
                    agent {
                        docker {
                            image 'node:18-alpine'
                        }
                    }
                    steps {
                        sh 'echo "With docker"'
                        sh 'npm --version'
                    }
                }
            }
        }
        
        ```
        
- Workspace Sync
    - `reuseNode true` - helps use the same workspace
        
        ```bash
        pipeline {
            agent any
        
            stages {
                stage('w/o docker') {
                    steps {
                        sh 'echo "Without docker"'
                    }
                }
        
                stage('w/ docker') {
                    agent {
                        docker {
                            image 'node:18-alpine'
                            reuseNode true
                        }
                    }
                    steps {
                        sh 'echo "With docker"'
                        sh 'npm --version'
                    }
                }
            }
        }
        ```
        
- Using GitRepo in Jenkins - Manually Trigger the pipeline build
    - Jenkinsfile
    - Jenkins → New Pipeline → Pipeline script from SCM
    - Git as SCM & Repository URL
    - Branch - `*/main`
    - ScriptPath - `{{$filename}}`
- Building the project - to Production
    - npm ci → designed for CI
        
        ```bash
        pipeline {
            agent any
        
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
                            ls -la
                            node --version
                            npm --version
                            npm ci
                            npm run build
                            ls -la
                        '''
                    }
                }
            }
        }
        ```
        
- Assignment 1 - Running Test
    - Look into build folder and check whether index.html is there
    - Execute tests that the project has
        - `npm test`
    - My Answer
        
        ```bash
        pipeline {
            agent any
        
            stages {
                stage('Test') {
                    agent {
                        docker {
                            image 'node:18-alpine'
                            reuseNode true
                        }
                    }
                    steps {
                        sh '''
                            echo 'Stage Test'
                            if [ -e build/index.html ]
                            then 
                                echo "exist"
                            else 
                                echo "not exist"
                            fi
                            echo 'Start running test'
                            npm ci
                            npm run build
                            npm test
                        '''
                    }
                }
            }
        }
        ```
        
- Publishing a JUnit Test Report
    - Test Result will be added as a section in the build result section
    
    ```bash
    pipeline {
        agent any
    
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
                        npm ci
                        npm run build
                        npm test
                    '''
                }
            }
            stage('Test') {
                agent {
                    docker {
                        image 'node:18-alpine'
                        reuseNode true
                    }
                }
                steps {
                    sh '''
                        echo 'Stage Test'
                        echo 'Start running test'
                        npm test
                    '''
                }
            }
        }
        post {
            always {
                junit 'test-results/junit.xml'
            }
        }
    }
    ```
    
- Comments in Jenkins
    - Within the `sh` block
        - `# comment`
    - Within step script
        - `// comment`  or `/* comment */`
- Run End-to-End Test
    - Tool - PlayWright (https://playwright.dev/docs/docker)
    - `npx playwrite test`
    - Use root privilege to run the code - **Please don’t do this**
        - `args `-u root:root``
            
            ```bash
            agent {
                docker {
                    image 'node:18-alpine'
                    reuseNode true
                    args `-u root:root`
                }
            }
            ```
            
    - Use HTML reporter
        - `npx playwrite test --report=html`
        - Need to enable this through Jenkins as well, otherwise, it will be all blank if you open this html file
            - `System.setProperty("hudson.model.DirectoryBrowserSupport.CSP", "sandbox allow-scripts;")`
            - Try to run this script in Jenkins Dashboard > Manage Jenkins > section “Tools and actions” > Script Console
        - You can also publish this html report so that you don’t need to locate this report under your workspace directory
            - To generate the code that helps generate the report, you can ask Jenkins for help
            - Your project → Config → Pipeline Syntax
                - Sample Step: publishHTML
                - HTML directory to archive: your output directory which contains that html file
                - index page: your actual report html file
    - Finished Version
        
        ```bash
        pipeline {
            agent any
        
            stages {
                stage('Build') {
                    agent {
                        docker {
                            image 'node:23-alpine'
                            reuseNode true
                        }
                    }
                    steps {
                        sh '''
                            npm ci
                            npm run build
                            npm test
                        '''
                    }
                }
                stage('Test') {
                    agent {
                        docker {
                            image 'node:23-alpine'
                            reuseNode true
                        }
                    }
                    steps {
                        sh '''
                            echo 'Stage Test'
                            echo 'Start running test'
                            npm test
                        '''
                    }
                }
                stage('ETE') {
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
                            npx playwright test
                        '''
                    }
                }
            }
        
            post {
                always {
                    script {
                        if (fileExists('jest-results/junit.xml')) {
                            junit 'jest-results/junit.xml'
                        } else {
                            echo 'JUnit report not found. Skipping test result publication.'
                        }
                    }
                }
            }
        }
        ```
        
- Run Jenkins in Parallel
    - Put the parallel stages within the parallel block under a parent stage
    - Example
        
        ```bash
        stage('Run Tests') {
            **parallel** {
                stage('Unit Tests') {
                    agent {
                        docker {
                            image 'node:23-alpine'
                            reuseNode true
                        }
                    }
                    steps {
                        sh '''
                            echo 'Stage Test'
                            echo 'Start running test'
                            npm test
                        '''
                    }
                    post {
                        always {
                            junit 'jest-results/junit.xml'
                        }
                    }
                }
                stage('ETE Test') {
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
                            publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, keepAll: false, reportDir: 'playwright-report', reportFiles: 'index.html', reportName: 'Playwright HTML Report', reportTitles: '', useWrapperFileDirectly: true])
                        }
                    }
                }
            }
        }
        ```
        
- Jenkins Plugin - Blue Ocean - Helps see the parallel process tree
    - However, I didn’t see the workspace folder which contains all output files.

## Implementing Continuous Deployment (CD) with Jenkins

- Tools - Netlify - for website hosting
- Manage Secrets using Jenkins
    - Jenkins Dashboard → Manage Jenkins → Credentials → System → Global Credentials
        - For Netlify, since that’s a token, we’ll use secrete text method
    - Use it within JenkinsFile - Example
        
        ```bash
          environment {
              NETLIFY_SITE_ID = '95604d86-4074-489a-ba85-0ffa341670b7'
              NETLIFY_AUTH_TOKEN = credentials('netlify-token')
          }
        ```
        
- Deploy to Production
    - Added the code that helps deploy the website within the steps stage
        
        ```bash
        stage('Deploy') {
            agent {
                docker {
                    image 'node:23-alpine'
                    reuseNode true
                }
            }
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
        ```
        
- Build periodically (Scheduled Builds)
    - Project → Configuration
    - Build Periodically
        - Cron job format - Example `H 3 * * 1-5` → Every Monday to Friday at 3 am. H is a place holder, helps execute job once a day but not all at the same time.
    - Git Hook
        - Github notify the Jenkins
    - Poll SCM
        - Jenkins check Github periodically
        - Specify the frequency of Jenkins checking Github Repo
        - At most every minute
        - You can see the pulling status through Git Polling Log
    - Post Deployment Tests - Use a staging environment
- Manual approval step before deploying to production
    - Add Another Stage to ask User’s Input
        - `input "xxxxx?"`
    - You can also specify the OK Button Caption
        - `input message: 'Ready to deploy?', ok: 'Yes, I am sure I want to deploy!'`
    - If there’s no user input, we can time out the build process
        
        ```bash
        timeout(time: 1, unit: 'HOURS') {
        	input message: 'Ready to deploy?', ok: 'Yes, I am sure I want to deploy!'
        }
        ```
        
- Assignment
    - Add an approval stage
- Parse Response Data
    - Reason: Netlify returns a random website URL that users can access to the application. Need this URL within our test stage
    - Example
        
        ```bash
        {
          "site_id": "95604d86-4074-489a-ba85-0ffa341670b7",
          "site_name": "capable-sunburst-3682c0",
          "deploy_id": "676c63d12c3980b100cbc538",
          "deploy_url": "https://676c63d12c3980b100cbc538--capable-sunburst-3682c0.netlify.app",
          "logs": "https://app.netlify.com/sites/capable-sunburst-3682c0/deploys/676c63d12c3980b100cbc538",
          "function_logs": "https://app.netlify.com/sites/capable-sunburst-3682c0/logs/functions?scope=deploy:676c63d12c3980b100cbc538",
          "edge_function_logs": "https://app.netlify.com/sites/capable-sunburst-3682c0/logs/edge-functions?scope=deployid:676c63d12c3980b100cbc538"
        }
        ```
        
    - Steps:
        - store the response returned from Netlify to a json file
        - Find out the value of a property using node-jq library within the file
- Parse dynamic data between stages
    - Need a script block
        
        ```bash
        pipeline {
            agent any
        
            stages {
                stage('Deploy') {
                    steps {
                        echo 'Deploying ...'
                        **script {
                            env.MY_VAR = sh(script: 'date', returnStdout: true)
                        }**
                    }
                }
        
                stage('E2E') {
                    steps {
                        echo 'E2E ...'
                        echo "MY_VAR is: ${env.MY_VAR}"
                        echo "MY_VAR is: $env.MY_VAR" // This syntax should work as well
                    }
                }
            }
        }
        
        ```
        
    - Add dynamic URL to the Jenkins File
        - Steps
            - Save the URL within the response to a local file
            - Use node-jq to read the variable and assign it to an environment variable
            - Within other stages, use these variables
        - Example
            
            ```bash
            pipeline {
                agent any
            
                environment {
                    NETLIFY_SITE_ID = '95604d86-4074-489a-ba85-0ffa341670b7'
                    NETLIFY_AUTH_TOKEN = credentials('netlify-token')
                }
            
                stages {
                    stage('Build') {
                        agent {
                            docker {
                                image 'node:23-alpine'
                                reuseNode true
                            }
                        }
                        steps {
                            sh '''
                                npm ci
                                npm run build
                                npm test
                            '''
                        }
                    }
            
                    stage('Run Tests') {
                        parallel {
                            stage('Unit Tests') {
                                agent {
                                    docker {
                                        image 'node:23-alpine'
                                        reuseNode true
                                    }
                                }
                                steps {
                                    sh '''
                                        echo 'Stage Test'
                                        echo 'Start running test'
                                        npm test
                                    '''
                                }
                                post {
                                    always {
                                        junit 'jest-results/junit.xml'
                                    }
                                }
                            }
                            stage('ETE Test') {
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
                                        publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, keepAll: false, reportDir: 'playwright-report', reportFiles: 'index.html', reportName: 'Playwright HTML Report', reportTitles: '', useWrapperFileDirectly: true])
                                    }
                                }
                            }
                        }
                    }
            
                    stage('Deploy Staging') {
                        agent {
                            docker {
                                image 'node:23-alpine'
                                reuseNode true
                            }
                        }
                        steps {
                            sh '''
                                npm install netlify-cli
                                npm install node-jq
                                node_modules/.bin/netlify --version
                                echo "Deploying to production. Site ID: $NETLIFY_SITE_ID"
                                node_modules/.bin/netlify status
                                node_modules/.bin/netlify deploy --dir=build --json > deploy-output.json
                                node_modules/.bin/node-jq -r '.deploy_url' deploy-output.json
                            '''
                            script {
                                env.STAGING_URL = sh(script: "node_modules/.bin/node-jq -r '.deploy_url' deploy-output.json", returnStdout: true)
                            }
                        }
                    }
            
                    stage('Staging ETE Test') {
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
                                publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, keepAll: false, reportDir: 'playwright-report', reportFiles: 'index.html', reportName: 'Playwright Staging E2E Report', reportTitles: '', useWrapperFileDirectly: true])
                            }
                        }
                    }
            
                    stage('Approval') {
                        steps {
                            timeout(1) {
                                input message: 'Approve to be deployed to production? ', ok: 'Yes'
                            }
                        }
                    }
                    stage('Deploy Prod') {
                        agent {
                            docker {
                                image 'node:23-alpine'
                                reuseNode true
                            }
                        }
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
                    stage('Prod ETE Test') {
                        agent {
                            docker {
                                image 'mcr.microsoft.com/playwright:v1.39.0-jammy'
                                reuseNode true
                            }
                        }
                        environment {
                            CI_ENVIRONMENT_URL = 'https://capable-sunburst-3682c0.netlify.app/'
                        }
                        steps {
                            sh '''
                                npx playwright test --reporter=html
                            '''
                        }
                        post {
                            always {
                                publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, keepAll: false, reportDir: 'playwright-report', reportFiles: 'index.html', reportName: 'Playwright Production E2E Report', reportTitles: '', useWrapperFileDirectly: true])
                            }
                        }
                    }
                }
            }
            ```
            
- Combining stages together
    - For example
        - Deploying Stage & Staging E2E Test together
        - Deploying Prod & Prod E2E Test together
        
        ```bash
        pipeline {
            agent any
        
            environment {
                NETLIFY_SITE_ID = 'YOUR NETLIFY SITE ID'
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
                                    npx playwright test  --reporter=html
                                '''
                            }
        
                            post {
                                always {
                                    publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, keepAll: false, reportDir: 'playwright-report', reportFiles: 'index.html', reportName: 'Local E2E', reportTitles: '', useWrapperFileDirectly: true])
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
                            npm install netlify-cli node-jq
                            node_modules/.bin/netlify --version
                            echo "Deploying to staging. Site ID: $NETLIFY_SITE_ID"
                            node_modules/.bin/netlify status
                            node_modules/.bin/netlify deploy --dir=build --json > deploy-output.json
                            CI_ENVIRONMENT_URL=$(node_modules/.bin/node-jq -r '.deploy_url' deploy-output.json)
                            npx playwright test  --reporter=html
                        '''
                    }
        
                    post {
                        always {
                            publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, keepAll: false, reportDir: 'playwright-report', reportFiles: 'index.html', reportName: 'Staging E2E', reportTitles: '', useWrapperFileDirectly: true])
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
        
                stage('Deploy prod') {
                    agent {
                        docker {
                            image 'mcr.microsoft.com/playwright:v1.39.0-jammy'
                            reuseNode true
                        }
                    }
        
                    environment {
                        CI_ENVIRONMENT_URL = 'YOUR NETLIFY URL'
                    }
        
                    steps {
                        sh '''
                            node --version
                            npm install netlify-cli
                            node_modules/.bin/netlify --version
                            echo "Deploying to production. Site ID: $NETLIFY_SITE_ID"
                            node_modules/.bin/netlify status
                            node_modules/.bin/netlify deploy --dir=build --prod
                            npx playwright test  --reporter=html
                        '''
                    }
        
                    post {
                        always {
                            publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, keepAll: false, reportDir: 'playwright-report', reportFiles: 'index.html', reportName: 'Prod E2E', reportTitles: '', useWrapperFileDirectly: true])
                        }
                    }
                }
            }
        }
        
        ```
        
- Continuous Delivery vs Continuous Deployment
    - Usually, Continuous Deployment has no Human Approval involved.

## Docker Review

- Build your own image (Through DockerFile)
    - Choose your base image
        - `FROM $Your_Image_Name`
    - Go through all other dependencies installation
        - `RUN npm install -g xxxxx`
    - Build the image - within your Jenkins steps
        - `sh 'docker build -f $DockerFileName -t $Your_Image_Name .'`  - This helps store the customized image in your local repo. In the future, if you want to use this image through Jenkins process, you
    - Push it to Docker Hub so that it can be reused later
        - Log into Docker
            - `docker login`
        - Tag the image
            - `docker tag your-image:latest your-username/your-repo-name:version-tag`
        - Push to the docker hub repo
            - `docker push your-username/your-repo-name:version-tag`
- Microsoft Official Images
    - Microsoft Artifact Registry (mcr.microsoft.com/en-us/)
- Assignment - Finished
- Another Jenkins Pipeline for image building only
    - To save time

## CD to AWS

- Basically, need to use AWS CLI within Jenkins
    
    ```bash
    stage('AWS') {
        agent {
            docker {
                image 'amazon/aws-cli'
                args "--entrypoint=''"
            }
        }
        steps {
            sh '''
                aws --version
                aws s3 ls
            '''
        }
    }
    
    ```
    
- Need to create an IAM User and allow the access to the S3
    - Need to have Access Key - for CLI
    - Save the Access Key & Access Secret within Jenkins Credential Management
    - To Generate Syntax, useJenkins Syntax Generator
        - Username and Password from the dropdown
        - Provide the access username and access password
        
        ```bash
        stage('AWS') {
            agent {
                docker {
                    image 'amazon/aws-cli'
                    args "--entrypoint=''"
                }
            }
            environment {
                AWS_S3_Bucket = "cd-learning-bucket-20241227"
            }
            steps {
                withCredentials([usernamePassword(credentialsId: 'aws-s3-key', passwordVariable: 'AWS_SECRET_ACCESS_KEY', usernameVariable: 'AWS_ACCESS_KEY_ID')]) {
                    sh '''
                        aws --version
                        echo "Hello S3!" > index.html
                        aws s3 cp index.html s3://$AWS_S3_Bucket/index.html
                    '''
                }
            }
        }
        ```
        

## CD to ECS

- Workflow
    - Build a Docker image with the app
    - Store the Docker image in a Docker Registry
    - Deploy the Docker container using Amazon Elastic Container Service (ECS)
        - Create the cluster - a group of servers that run containerized applications
        - Create task definition - Blue Print of the task
        - Run the task definition
- ECS
    - EC2 Instance - Need self-management of the server
    - Fargate (serverless) - Not Free
        - Update ECS Task Definition through Jenkins
            - File Example
                
                ```bash
                pipeline {
                    agent any
                
                    environment {
                        AWS_DEFAULT_REGION = "us-east-1"
                    }
                
                    stages {
                        stage('Deploy to AWS') {
                            agent {
                                docker {
                                    image 'amazon/aws-cli'
                                    reuseNode true
                                    args "--entrypoint=''"
                                }
                            }
                            steps {
                                withCredentials([usernamePassword(credentialsId: 'aws-s3-key', passwordVariable: 'AWS_SECRET_ACCESS_KEY', usernameVariable: 'AWS_ACCESS_KEY_ID')]) {
                                    sh '''
                                        aws --version
                                        aws ecs register-task-definition --cli-input-json file://aws/task-definition-prod.json
                                    '''
                                }
                            }
                        }
                        stage('Build') {
                            agent {
                                docker {
                                    image 'node:23-alpine'
                                    reuseNode true
                                }
                            }
                            steps {
                                sh '''
                                    npm ci
                                    npm run build
                                    npm test
                                '''
                            }
                        }
                    }
                }
                ```
                
            - Remember to set up the permission of the AWS user so that the user has access to ECS
            - Remember to add your updated task definition json file to your current workspace and locate the relative path of the file
        - Update ECS Service through Jenkins
            - File Example
                - Remember to specify the cluster by using command `--cluster $clustername`
                - Remember to specify the service by using the command `--service $servicename`
                - Remember to specify the task definition name & revision version by using the command `--task-definition $taskdefinitionname:revision`
                    
                    ```bash
                    pipeline {
                        agent any
                    
                        environment {
                            AWS_DEFAULT_REGION = "us-east-1"
                        }
                    
                        stages {
                            stage('Deploy to AWS') {
                                agent {
                                    docker {
                                        image 'amazon/aws-cli'
                                        reuseNode true
                                        args "--entrypoint=''"
                                    }
                                }
                                steps {
                                    withCredentials([usernamePassword(credentialsId: 'aws-s3-key', passwordVariable: 'AWS_SECRET_ACCESS_KEY', usernameVariable: 'AWS_ACCESS_KEY_ID')]) {
                                        sh '''
                                            aws --version
                                            aws ecs register-task-definition --cli-input-json file://aws/task-definition-prod.json
                                            aws ecs update-service --cluster LearnJenkinsApp-Cluster --service LearnJenkinsApp-Service-Prod --task-definition LearnJenkinsApp-TaskDefinition-Prod:2
                                        '''
                                    }
                                }
                            }
                            stage('Build') {
                                agent {
                                    docker {
                                        image 'node:23-alpine'
                                        reuseNode true
                                    }
                                }
                                steps {
                                    sh '''
                                        npm ci
                                        npm run build
                                        npm test
                                    '''
                                }
                            }
                        }
                    }
                    ```
                    
                - We can also use node-jq to read latest version of the task-definition dynamically - Example showing below
                    
                    ```bash
                    pipeline {
                        agent any
                    
                        environment {
                            AWS_DEFAULT_REGION = "us-east-1"
                        }
                    
                        stages {
                            stage('Deploy to AWS') {
                                agent {
                                    docker {
                                        image 'amazon/aws-cli'
                                        reuseNode true
                                        args "-u root --entrypoint=''"
                                    }
                                }
                                steps {
                                    withCredentials([usernamePassword(credentialsId: 'aws-s3-key', passwordVariable: 'AWS_SECRET_ACCESS_KEY', usernameVariable: 'AWS_ACCESS_KEY_ID')]) {
                                        sh '''
                                            aws --version
                                            yum install jq -y
                                            LATEST_TD_REVISION=$(aws ecs register-task-definition --cli-input-json file://aws/task-definition-prod.json | jq '.taskDefinition.revision')
                                            echo $LATEST_TD_REVISION
                                            aws ecs update-service --cluster LearnJenkinsApp-Cluster --service LearnJenkinsApp-Service-Prod --task-definition LearnJenkinsApp-TaskDefinition-Prod:$LATEST_TD_REVISION
                                        '''
                                    }
                                }
                            }
                            stage('Build') {
                                agent {
                                    docker {
                                        image 'node:23-alpine'
                                        reuseNode true
                                    }
                                }
                                steps {
                                    sh '''
                                        npm ci
                                        npm run build
                                        npm test
                                    '''
                                }
                            }
                        }
                    }
                    ```
                    
    - Wait Command
        - Reason: it takes some time to stop the current task & service and run the new task & service. Thus, use AWS’ wait command to make it work more smoothly
        - To pause running until a service is confirmed to be stable
            - `aws ecs wait services-stable -- cluster $clustername --services $myservice`
- ECR - Use ECR to store customized image
    - Install docker within AWS CLI Image
        - `amazon-linux-extras install docker`
    - Map Docker Daemon Socket so that container can connect to it
        - `args "-u root **-v /var/run/docker.sock:/var/run/docker.sock** --entrypoint=''"`
    - Push new image to ECR
        
        ```bash
        steps {
            withCredentials([usernamePassword(credentialsId: 'aws-s3-key', passwordVariable: 'AWS_SECRET_ACCESS_KEY', usernameVariable: 'AWS_ACCESS_KEY_ID')]) {
                sh '''
                    docker build -f DockerFile-ECR -t $AWS_ECR_REGISTRY/$APP_NAME:1.0 .
                    aws ecr get-login-password | docker login --username AWS --password-stdin $AWS_ECR_REGISTRY
                    docker push $AWS_ECR_REGISTRY/$APP_NAME:1.0
                '''
            }
        }
        ```
        
- Assign this image to ECS Task Definition
    - Update original Task-Definition-Json File
        - Image - will be the image name within ECR
        - ExecutionRoleArn - can be obtained by manually run an image task through ECS
        
        ```bash
        {
            "requiresCompatibilities": [
                "FARGATE"
            ],
            "family": "LearnJenkinsApp-TaskDefinition-Prod",
            "containerDefinitions": [
                {
                    "name": "learnjenkinsapp",
                    "image": "284844911629.dkr.ecr.us-east-1.amazonaws.com/learningcicd:1.0",
                    "portMappings": [{
                            "name":"nginx-80-tip",
                            "containerPort": 80,
                            "hostPort": 80,
                            "protocol": "tcp",
                            "appProtocol": "http"
                        }],
                    "essential": true
                }
            ],
            "volumes": [],
            "networkMode": "awsvpc",
            "memory": "512",
            "cpu": "256",
            "executionRoleArn": "arn:aws:iam::284844911629:role/ecsTaskExecutionRole"
        }
        ```
        
    - To make this run on dynamically updated app version, we can use a tool from linux called sed
        - within the task definition task file, name the app version with a unique name - Example `#APP_VERSION#`
            
            ```bash
            {
                "requiresCompatibilities": [
                    "FARGATE"
                ],
                "family": "LearnJenkinsApp-TaskDefinition-Prod",
                "containerDefinitions": [
                    {
                        "name": "learnjenkinsapp",
                        "image": "284844911629.dkr.ecr.us-east-1.amazonaws.com/learningcicd:#APP_VERSION#",
                        "portMappings": [{
                                "name":"nginx-80-tip",
                                "containerPort": 80,
                                "hostPort": 80,
                                "protocol": "tcp",
                                "appProtocol": "http"
                            }],
                        "essential": true
                    }
                ],
                "volumes": [],
                "networkMode": "awsvpc",
                "memory": "512",
                "cpu": "256",
                "executionRoleArn": "arn:aws:iam::284844911629:role/ecsTaskExecutionRole"
            }
            ```
            
        - within the JenkinsFile where the image was built, assign the correct app version value to this variable
            - `sed -i "s#APP_VERSION#/$REACT_APP_VERSION/g" aws/task-definition-prod.json`
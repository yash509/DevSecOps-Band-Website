pipeline {
    agent any
    tools {
        jdk 'jdk17'
        nodejs 'node16'
    }
    environment {
        SCANNER_HOME = tool 'sonar-scanner'
    }
    stages {
        stage('clean workspace') {
            steps {
                cleanWs()
            }
        }
        stage('Checkout from Git') {                        
            steps {                                       
                git branch: 'main', url: 'https://github.com/yash509/Web-dev-projects.git'
            }
        }
        stage('Deployments') {
            parallel {
                stage('deploy to staging') {
                    steps {
                        echo 'staging deployment done'
                    }
                }
                stage('deploy to production') {
                    steps {
                        echo 'production deployment done'
                    }
                }
            }
        }
        stage('Test Build') {
            steps {
                echo 'Building....'
            }
            post {
                always {
                    jiraSendBuildInfo site: 'clouddevopshunter.atlassian.net'
                }
            }
        }
        stage('Deploy - Staging') {
            when {
                branch 'master'
            }
            steps {
                echo 'Deploying to Staging from main....'
            }
            post {
                always {
                    jiraSendDeploymentInfo environmentId: 'us-stg-1', environmentName: 'us-stg-1', environmentType: 'staging'
                }
            }
        }
        stage('Deploy - Production') {
            when {
                branch 'master'
            }
            steps {
                echo 'Deploying to Production from main....'
            }
            post {
                always {
                    jiraSendDeploymentInfo environmentId: 'us-prod-1', environmentName: 'us-prod-1', environmentType: 'production'
                }
            }
        }
        stage("Sonarqube Analysis ") {                         
            steps {
                dir('Band Website') {
                    withSonarQubeEnv('sonar-server') {
                    sh '''$SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=band \
                    -Dsonar.projectKey=band'''
                    }
                }
            }
        }
        stage("quality gate") {
            steps {
                dir('Band Website') {
                    script {
                        waitForQualityGate abortPipeline: false, credentialsId: 'Sonar-token'
                    }
                }
            }
        }
        stage('Install Dependencies') {
            steps {
                dir('Band Website') {
                    sh "npm install"
                }
            }
        }
        stage('OWASP FS SCAN') {
            steps {
                dir('Band Website') {
                    dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit', odcInstallation: 'DP-Check'
                    dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
                }
            }
        }
        stage('TRIVY FS SCAN') {
            steps {
                dir('Band Website') {
                    sh "trivy fs . > trivyfs.txt"
                }
            }
        }
        stage("Docker Image Building"){
            steps{
                script{
                    dir('Band Website') {
                        withDockerRegistry(credentialsId: 'docker', toolName: 'docker'){   
                            sh "docker build -t band-web ." 
                            
                        }
                    }
                }
            }
        }
        stage("Docker Image Tagging"){
            steps{
                script{
                    dir('Band Website') {
                        withDockerRegistry(credentialsId: 'docker', toolName: 'docker'){
                            sh "docker tag band-web yash5090/band-web:latest " 
                        }
                    }
                }
            }
        }
        stage("Image Push to DockerHub") {
            steps{
                script{
                    dir('Band Website') {
                        withDockerRegistry(credentialsId: 'docker', toolName: 'docker') {
                            sh "docker push yash5090/band-web:latest "
                        }
                    }
                }
            }
        }
        stage("TRIVY"){
            steps{
                dir('Band Website') {
                    sh "trivy image yash5090/band-web:latest > trivyimage.txt"   
                }
            }
        }
        stage ('Manual Approval'){
          steps {
           script {
             timeout(time: 10, unit: 'MINUTES') {
              approvalMailContent = """
              Project: ${env.JOB_NAME}
              Build Number: ${env.BUILD_NUMBER}
              Go to build URL and approve the deployment request.
              URL de build: ${env.BUILD_URL}
              """
             mail(
             to: 'clouddevopshunter@gmail.com',
             subject: "${currentBuild.result} CI: Project name -> ${env.JOB_NAME}", 
             body: approvalMailContent,
             mimeType: 'text/plain'
             )
            input(
            id: "DeployGate",
            message: "Deploy ${params.project_name}?",
            submitter: "approver",
            parameters: [choice(name: 'action', choices: ['Deploy'], description: 'Approve deployment')])  
            }
           }
          }
        }
        stage('Deploy to container'){
            steps{
                dir('BMI Calculator (JS)') {
                    sh 'docker run -d --name band-web -p 5000:80 yash5090/band-web:latest' 
                }
            }
        }
        stage('Deployment Done') {
            steps {
                echo 'Deployed Succcessfully...'
            }
        }
    }
     post { 
    always { 
        script { 
            def jobName = env.JOB_NAME 
            def buildNumber = env.BUILD_NUMBER 
            def pipelineStatus = currentBuild.result ?: 'UNKNOWN' 
            def bannerColor = pipelineStatus.toUpperCase() == 'SUCCESS' ? 'green' : 'red' 
 
            def body = """ 
                <html> 
                <body> 
                <div style="border: 4px solid ${bannerColor}; padding: 10px;"> 
                <h2>${jobName} - Build ${buildNumber}</h2> 
                <div style="background-color: ${bannerColor}; padding: 10px;"> 
                <h3 style="color: white;">Pipeline Status: ${pipelineStatus.toUpperCase()}</h3> 
                </div> 
                <p>Check the <a href="${BUILD_URL}">console output</a>.</p> 
                </div> 
                </body> 
                </html> 
            """ 
 
            emailext ( 
                subject: "${jobName} - Build ${buildNumber} - ${pipelineStatus.toUpperCase()}", 
                body: body, 
                to: 'clouddevopshunter@gmail.com', 
                from: 'jenkins@example.com', 
                replyTo: 'jenkins@example.com', 
                mimeType: 'text/html', 
                attachmentsPattern: 'trivy-image-report.html,trivyfs.txt,trivyimage.txt') 
            } 
        } 
    }
    post {
        always {
            emailext attachLog: true,
            subject: "'${currentBuild.result}'",
            body: "Project: ${env.JOB_NAME}<br/>" +
            "Build Number: ${env.BUILD_NUMBER}<br/>" + 
            "URL: ${env.BUILD_URL}<br/>",
            to: 'clouddevopshunter@gmail.com',
            attachmentsPattern: 'trivyfs.txt,trivyimage.txt'
        }
    }
}
stage('Result') {
        timeout(time: 10, unit: 'MINUTES') {
            mail to: 'clouddevopshunter@gmail.com',
            subject: "${currentBuild.result} CI: ${env.JOB_NAME}",
            body: "Project: ${env.JOB_NAME}\nBuild Number: ${env.BUILD_NUMBER}\nGo to ${env.BUILD_URL} and approve deployment"
            input message: "Deploy ${params.project_name}?", 
            id: "DeployGate", 
            submitter: "approver", 
            parameters: [choice(name: 'action', choices: ['Success'], description: 'Approve deployment')]
        }
    }

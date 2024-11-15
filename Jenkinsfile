pipeline {
    agent any

    tools{
        jdk 'JDK 17'
        maven 'Maven 3.8.1'
    }

    environment {
        SCANNER_HOME= tool ('sonar-scanner')
        IMAGE_TAG= arunagri03/boardgame
    }

    stages{
        stage('git checkout') {
            steps{
                git branch: 'main', credentialsId: 'github_token', url: 'https://github.com/arun037/boardgame-jenkins.git'
            }
        }

        stage('compile') {
            steps{
                sh 'mvn compile'  
            }
        }

        stage('Test') {
            steps{
                sh 'mvn test'

            }
        }

        stage('file system scan') {
            steps{
                sh 'trivy fs --format table -o trivy-fs-report.html .'
            }
        }

        stage('sonarqube scan') {
            steps{
                withSonarQubeEnv('sonar') {
                    sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=BoardGame -Dsonar.projectKey=BoardGame \
                            -Dsonar.java.binaries=. '''

              }
            }
        }

        stage('Quality gate') {
            steps{
                 waitForQualityGate abortPipeline: false, credentialsId: 'sonar-token'
            }
        }

        stage('Build'){
            steps{
                sh 'mvn package'
            }
        }

        stage('publish to Nexus repository'){
            steps{
                withMaven(globalMavenSettingsConfig: 'global-settings', jdk: 'JDK 11', maven: 'Maven 3.8.1', mavenSettingsConfig: '', traceability: true) {
                    sh "mvn deploy"
                }
            }
        }

        stage('docker build and tag'){
            steps{
                script {
                    withDockerRegistry(credentialsId: 'docker-token', toolName: 'docker') {
                        sh 'docker build -t $IMAGE_TAG .'
                    }
                }
            }
        }

        stage('Docker image scan'){
            steps{
                sh 'trivy image --format table -o trivy-image-report.html $IMAGE_TAG'
            }
        }

        stage('push image to hub'){
            steps{
                sh 'docker push $IMAGE_TAG'
            }
        }

        stage('deploy to eks'){
            steps{
                withAWS(credentials: 'aws_id') {
                    sh 'aws eks update-kubeconfig --region ap-south-1 --name my-cluster'
                    sh 'kubectl apply -f deployment-service.yaml'
                }
            }
        }

        stage('verifying the created deployment'){
            steps{
                sh 'kubectl get pods'
            }
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
                to: 'arunagri03@gmail.com',
                from: 'jenkins@example.com',
                replyTo: 'jenkins@example.com',
                mimeType: 'text/html',
                attachmentsPattern: 'trivy-image-report.html'
            )
        }
    }
}

}

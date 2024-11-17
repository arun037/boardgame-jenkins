pipeline {
    agent any

    tools {
        jdk 'JDK 11'
        maven 'Maven 3.8.1'
    }

    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        IMAGE_TAG = 'arunagri03/boardgame'
    }

    stages {
        stage('Git Checkout') {
            steps {
                git branch: 'main', credentialsId: 'github_token', url: 'https://github.com/arun037/boardgame-jenkins.git'
            }
        }

        stage('Compile') {
            steps {
                sh 'mvn compile'
            }
        }

        stage('Test') {
            steps {
                sh 'mvn test'
            }
        }

        stage('Filesystem Scan') {
            steps {
                sh 'trivy fs --format table -o trivy-fs-report.html .'
            }
        }

        stage('SonarQube Scan') {
            steps {
                withSonarQubeEnv('sonar') {
                    sh '''$SCANNER_HOME/bin/sonar-scanner \
                          -Dsonar.projectName=BoardGame \
                          -Dsonar.projectKey=BoardGame \
                          -Dsonar.java.binaries=.'''
                }
            }
        }

        stage('Quality Gate') {
            steps {
                waitForQualityGate abortPipeline: false, credentialsId: 'sonar-token'
            }
        }

        stage('Build') {
            steps {
                sh 'mvn package'
            }
        }

        stage('Publish to Nexus Repository') {
            steps {
                withMaven(globalMavenSettingsConfig: 'global-settings', jdk: 'JDK 11', maven: 'Maven 3.8.1') {
                    sh "mvn deploy"
                }
            }
        }

        stage('Docker Build and Tag') {
            steps {
                script {
                    docker.build("$IMAGE_TAG")
                }
            }
        }

        stage('Image Scan') {
            steps {
                sh 'trivy image --format table -o trivy-image-report.html $IMAGE_TAG'
            }
        }

        stage('Docker Image Push') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker_token') {
                        sh 'docker push $IMAGE_TAG'
                    }
                }
            }
        }

        stage('Deploy to EKS') {
            steps {
                withKubeConfig(
                    credentialsId: 'k8-cred',
                    namespace: 'webapps',
                    serverUrl: 'https://172.31.45.11:6443'
                ) {
                    sh 'kubectl apply -f deployment-service.yaml'
                }
            }
        }

        stage('Verify Deployments') {
            steps {
                withKubeConfig(
                    credentialsId: 'k8-cred',
                    namespace: 'webapps',
                    serverUrl: 'https://172.31.45.11:6443'
                ) {
                    sh 'kubectl get pods -n webapps'
                    sh 'kubectl get svc -n webapps'
                }
            }
        }
    }

    post {
        always {
            emailext(
                attachLog: true,
                subject: "'Build ${currentBuild.result}: ${env.JOB_NAME} #${env.BUILD_NUMBER}'",
                body: """<p>Project: ${env.JOB_NAME}</p>
                         <p>Build Number: ${env.BUILD_NUMBER}</p>
                         <p>URL: <a href="${env.BUILD_URL}">${env.BUILD_URL}</a></p>""",
                to: 'arunagri03@gmail.com',
                attachmentsPattern: 'trivy-fs-report.html,trivy-image-report.html'
            )
        }
    }
}

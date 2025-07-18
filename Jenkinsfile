
pipeline {
    agent any

    environment {
        // Define credentials from Jenkins Credentials Manager
        DOCKER_CREDENTIALS = credentials('dockerhub-credentials-id') // Replace with your DockerHub credentials ID
        SONAR_TOKEN = credentials('sonarqube-token-id') // Replace with your SonarQube token credentials ID
        AWS_ACCOUNT_ID = credentials('aws-ID')
        AWS_REGION = 'eu-central-1' // Match your Terraform region
        ECR_REPO_NAME = 'spring-app'
        DOCKER_IMAGE_NAME = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO_NAME}"
        KUBE_CONFIG = credentials('kubeconfig') // Assumes you've added your kubeconfig as a file credential
    }

    tools {
        maven 'Maven3'
    }

    stages {
        stage('1. Checkout Code') {
            steps {
                git branch: 'main', url: 'https://github.com/Suxita/CI-CD-with-jenkins'
            }
        }

        stage('2. Build Application') {
            steps {
                sh 'mvn clean package'
            }
        }

        stage('3. Unit Tests') {
            steps {
                sh 'mvn test'
            }
            post {
                always {
                    junit 'target/surefire-reports/*.xml'
                }
            }
        }

        stage('4. SonarQube Security Scan') {
            steps {
                withSonarQubeEnv('SonarQube') { // Assumes 'SonarQube' is configured in Jenkins System Config
                    sh """
                        mvn sonar:sonar \
                        -Dsonar.projectKey=spring-app \
                        -Dsonar.host.url=http://18.194.239.38:9000 \
                        -Dsonar.login=${SONAR_TOKEN}
                    """
                }
            }
        }

        stage('5. Build & Push Docker Image') {
            steps {
                script {
                    def imageTag = "${DOCKER_IMAGE_NAME}:${env.BUILD_NUMBER}"
                    // Login to AWS ECR
                    sh "aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
                    
                    // Build and Push Image
                    docker.build(imageTag)
                    docker.push(imageTag)
                    docker.push("${DOCKER_IMAGE_NAME}:latest")
                }
            }
        }

        stage('6. Deploy to K8s with Helm') {
            steps {
                script {
                    // Use kubeconfig from credentials
                    withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG_FILE')]) {
                        sh "export KUBECONFIG=${KUBECONFIG_FILE}"
                        sh """
                            helm upgrade --install spring-app ./helm \
                                --namespace default \
                                --set image.repository=${DOCKER_IMAGE_NAME} \
                                --set image.tag='latest' \
                                --wait
                        """
                    }
                }
            }
        }
        
       stage('7. Application Verification') {
            steps {
                sleep time: 30, unit: 'SECONDS'
                script {
                    def appUrl = sh(script: 'minikube service spring-app --url', returnStdout: true).trim()
                    
                    if (!appUrl.startsWith("http")) {
                        appUrl = "http://" + appUrl
                    }

                    sh "curl --fail --silent ${appUrl}"
                    echo "Application is running at ${appUrl}"
                }
            }
        }
    }

    post {
        always {
            script {
                def jobStatus = currentBuild.result ?: 'SUCCESS'
                def jobName = env.JOB_NAME
                def buildNumber = env.BUILD_NUMBER
                def buildUrl = env.BUILD_URL
                def subject = "${jobStatus}: Jenkins Job '${jobName}' #${buildNumber}"
                def body = """
                    <p>Job: <a href='${buildUrl}'>${jobName}</a> #${buildNumber}</p>
                    <p>Status: <strong>${jobStatus}</strong></p>
                """
                emailext (
                    subject: subject,
                    body: body,
                    to: 'mishosukhishvili@gmail.com', // Replace with your email
                    mimeType: 'text/html'
                )
            }
        }
    }
}
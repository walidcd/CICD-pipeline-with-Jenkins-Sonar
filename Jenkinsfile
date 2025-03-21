pipeline {
    agent any

    environment {
        SONARQUBE_SERVER = 'sonar-server'  
        GITHUB_REPO = 'https://github.com/walidcd/CICD-pipeline-with-Jenkins-Sonar.git'
        SONAR_PROJECT_KEY = credentials('sonar-project') 
        SONARQUBE_TOKEN = credentials('sonar-token')
        NVD_API_KEY = credentials('NVD-API')
        DOCKERHUB_CREDENTIALS = "docker-hub-credentials-id" 
        DOCKER_IMAGE_NAME = 'walidboutahar/spring'
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', url: "${GITHUB_REPO}"
            }
        }

        stage('Build') {
            steps {
                sh 'chmod +x mvnw'
                sh './mvnw clean install'
            }
        }

        stage('Test') {
            steps {
                sh './mvnw test'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                script {
                    withSonarQubeEnv("${SONARQUBE_SERVER}") {
                        sh "./mvnw sonar:sonar " +
                           "-Dsonar.projectKey=${SONAR_PROJECT_KEY} " +
                           "-Dsonar.login=${SONARQUBE_TOKEN}"
                    }
                }
            }
        }

        stage('OWASP Dependency Check Scan and Publish') {
            steps {
                script {
                    dependencyCheck additionalArguments: """
                    -o './'
                    -s './'
                    -f 'ALL'
                    --prettyPrint""",
                    odcInstallation: "dependency-check"

                    dependencyCheckPublisher pattern: 'dependency-check-report.html'
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    sh "docker build -t ${DOCKER_IMAGE_NAME}:${env.BUILD_NUMBER} ."
                }
            }
        }

        stage('Trivy Scan') {
            steps {
                script {
                    def trivyHtmlReportFile = "trivy-report-${env.BUILD_NUMBER}.html"
                    sh """
                        trivy image --format template --template @/usr/local/share/trivy/templates/html.tpl \
                        ${DOCKER_IMAGE_NAME}:${env.BUILD_NUMBER} > ${trivyHtmlReportFile}
                    """
                    publishHTML([
                        reportName: 'Trivy Security Scan',
                        reportDir: '',
                        reportFiles: trivyHtmlReportFile,
                        keepAll: true
                    ])
                }
            }
        }

        stage('Push to DockerHub') {
            steps {
                script {
                    withDockerRegistry([credentialsId: "docker-hub-credentials-id", url: "https://index.docker.io/v1/"]) {
                        sh "docker push ${DOCKER_IMAGE_NAME}:${env.BUILD_NUMBER}"
                    }
                }
            }
        }
    }
}
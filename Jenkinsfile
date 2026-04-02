pipeline {
    agent any
    
    parameters {
        string(name: 'MAJOR_VERSION', defaultValue: '1', description: 'Major version for the Docker image tag (e.g., 1 for v1.x)')
    }
    
    tools {
        jdk 'jdk21'
        nodejs 'node20'
    }
    
    environment {
        SCANNER_HOME = tool 'sonarqube-scanner'
        TRIVY_HOME = '/usr/bin'
        REPO_URL = 'https://github.com/saw-cloudops/Netflix_Clone_On_Kubernetes__with_Jenkins-Security-Check-with-SonarQube.git'
        REPO_BRANCH = 'main'
        DOCKER_IMAGE_NAME = 'saw-cloudops/netflix'
        SONAR_PROJECT_NAME = 'Netflix'
        SONAR_PROJECT_KEY = 'Netflix'
        DOCKER_CREDENTIALS_ID = 'dockerhub'
        SONAR_CREDENTIALS_ID = 'SonarQube-Token'
        KUBERNETES_CREDENTIALS_ID = 'kubernetes'
        SONAR_SERVER = 'SonarQube-Server'
        DOCKER_TOOL_NAME = 'docker'
        TRIVY_FS_REPORT = 'trivyfs.txt'
        TRIVY_IMAGE_REPORT = 'trivyimage.txt'
        K8S_NAMESPACE = 'default'
        APP_NAME = 'netflix'
    }
    
    stages {
        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
            post {
                always {
                    echo "Workspace cleaned successfully"
                }
            }
        }
        stage('Checkout from Git') {
            steps {
                git branch: "${REPO_BRANCH}", url: "${REPO_URL}", credentialsId: 'github-credentials'
            }
        }
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv(credentialsId: "${SONAR_CREDENTIALS_ID}", installationName: "${SONAR_SERVER}") {
                    sh """
                    ${SCANNER_HOME}/bin/sonar-scanner \
                        -Dsonar.projectName=${SONAR_PROJECT_NAME} \
                        -Dsonar.projectKey=${SONAR_PROJECT_KEY}
                    """
                }
            }
        }
        stage('Quality Gate') {
            steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: "${SONAR_CREDENTIALS_ID}"
                }
            }
        }
        stage('Install Dependencies') {
            steps {
                script {
                    def nodeHome = tool name: 'node20', type: 'nodejs'
                    env.PATH = "${nodeHome}/bin:${env.PATH}"
                    sh "node --version"
                    sh "npm --version"
                    sh "npm install"
                }
            }
            post {
                failure {
                    echo "Failed to install dependencies. Check Node.js setup or package.json."
                }
            }
        }
        // stage('OWASP Dependency Check') {
        //     steps {
        //         withCredentials([string(credentialsId: 'nvd-api-key', variable: 'NVD_API_KEY')]) {
        //             dependencyCheck additionalArguments: "--scan ./ --disableYarnAudit --disableNodeAudit --nvdApiKey ${NVD_API_KEY}", odcInstallation: 'DP-Check'
        //             dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
        //         }
        //     }
        // }
        stage('Trivy FS Scan') {
            steps {
                script {
                    sh "${TRIVY_HOME}/trivy fs . > ${TRIVY_FS_REPORT}"
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: "${TRIVY_FS_REPORT}", allowEmptyArchive: true
                }
            }
        }
        stage('Set Version') {
            steps {
                script {
                    def buildNumber = env.BUILD_NUMBER ?: '0'
                    env.IMAGE_TAG = "v${params.MAJOR_VERSION}.${buildNumber}"
                    echo "Docker image will be tagged as: ${DOCKER_IMAGE_NAME}:${env.IMAGE_TAG}"
                }
            }
        }
        stage('Docker Build & Push') {
            steps {
                withCredentials([string(credentialsId: 'tmdb-api-key', variable: 'TMDB_API_KEY')]) {
                    script {
                        withDockerRegistry(credentialsId: "${DOCKER_CREDENTIALS_ID}", toolName: "${DOCKER_TOOL_NAME}") {
                            sh "docker build -t ${APP_NAME} --build-arg TMDB_V3_API_KEY=${TMDB_API_KEY} ."
                            sh "docker tag ${APP_NAME} ${DOCKER_IMAGE_NAME}:${env.IMAGE_TAG}"
                            sh "docker push ${DOCKER_IMAGE_NAME}:${env.IMAGE_TAG}"
                        }
                    }
                }
            }
            post {
                always {
                    sh "docker rmi ${APP_NAME} ${DOCKER_IMAGE_NAME}:${env.IMAGE_TAG} || true"
                }
            }
        }
        stage('Trivy Image Scan') {
            steps {
                script {
                    sh "${TRIVY_HOME}/trivy image ${DOCKER_IMAGE_NAME}:${env.IMAGE_TAG} > ${TRIVY_IMAGE_REPORT}"
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: "${TRIVY_IMAGE_REPORT}", allowEmptyArchive: true
                }
            }
        }
        stage('Deploy to Kubernetes') {
            steps {
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'aws-secret'
                ]]) {
                    script {
                        dir('Kubernetes') {
                            withKubeConfig(
                                credentialsId: "${KUBERNETES_CREDENTIALS_ID}",
                                serverUrl: 'https://BB493E81FD87EF61BB3DB644BFC7D606.yl4.ap-southeast-7.eks.amazonaws.com', // Replace with actual EKS endpoint
                                namespace: "${K8S_NAMESPACE}"
                            ) {
                                sh 'kubectl version'
                                sh "sed -i 's|image: ${DOCKER_IMAGE_NAME}:.*|image: ${DOCKER_IMAGE_NAME}:${env.IMAGE_TAG}|' deployment.yml"
                                sh 'kubectl apply -f deployment.yml'
                                sh 'kubectl apply -f service.yml'
                            }
                        }
                    }
                }
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
                to: 'saw-cloudops@gmail.com', // Update with your email
                attachmentsPattern: 'trivyfs.txt,trivyimage.txt'
        }
    }
}

pipeline {
    agent any
    tools {
        jdk 'jdk17'
        nodejs 'node16'
    }
    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        APP_NAME = "netflix-clone-pipeline"
        RELEASE = "1.0.0"
        DOCKER_USER = "onyima101"
        DOCKER_PASS = 'dockerhub'
        IMAGE_NAME = "${DOCKER_USER}/${APP_NAME}"
        IMAGE_TAG = "${RELEASE}-${BUILD_NUMBER}"
	    JENKINS_API_TOKEN = credentials("JENKINS_API_TOKEN")        
    }
    stages {
        stage('clean workspace') {
            steps {
                cleanWs()
            }
        }
        stage('Checkout from Git') {
            steps {
                git branch: 'main', credentialsId: 'github', url: 'https://github.com/onyima101/netflix-clone.git'
            }
        }
        stage("Sonarqube Analysis") {
            steps {
                withSonarQubeEnv('Sonarqube-Server') {
                    sh '''$SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=netflix \
                    -Dsonar.projectKey=netflix'''
                }
            }
        }
        stage("Quality Gate") {
            steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'Sonarqube-Token'
                }
            }
        }
        stage('Install Dependencies') {
            steps {
                sh "npm install"
            }
        }
        stage('OWASP FS SCAN') {
            steps {
                dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit', odcInstallation: 'DP-Check'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }
        stage('TRIVY FS SCAN') {
             steps {
                 sh "trivy fs . > trivyfs.txt"
            }
        }
        stage("Build & Push Docker Image") {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'dockerhub', toolName: 'docker') {
                        // sh "docker build --build-arg TMDB_V3_API_KEY=c62f51dd24dd41fd96569f8931ee34b7 -t netflix ."
                        sh "docker build -t netflix ."
                        sh "docker tag netflix ${DOCKER_USER}/${APP_NAME}:${IMAGE_TAG}"
                        sh "docker push ${DOCKER_USER}/${APP_NAME}:${IMAGE_TAG}"
                    }
                }
            }
        }              
        stage("TRIVY Image Scan"){
            steps{
                sh "trivy image ${DOCKER_USER}/${APP_NAME}:${IMAGE_TAG} > trivyimage.txt" 
            }
        }
        stage ('Cleanup Artifacts') {
            steps {
                script {
                    sh "docker rmi ${IMAGE_NAME}:${IMAGE_TAG}"
                }
            }
        }
	    stage("Trigger CD Pipeline") {
            steps {
                script {
                    sh "curl -v -k --user admin:${JENKINS_API_TOKEN} -X POST -H 'cache-control: no-cache' -H 'content-type: application/x-www-form-urlencoded' --data 'IMAGE_TAG=${IMAGE_TAG}' 'ec2-35-172-89-171.compute-1.amazonaws.com:8080/job/netflix-project-CD/buildWithParameters?token=gitops-token'"
                }
            }
        }
        stage("Check and Remove Existing Container") {
            steps {
                script {
                    def containerExists = sh(script: "docker ps -a -q -f name=${APP_NAME}", returnStdout: true).trim()
                    
                    if (containerExists) {
                        // Stop the container if it's running
                        sh "docker stop ${APP_NAME} || true"
                        
                        // Remove the container
                        sh "docker rm ${APP_NAME}"
                    } else {
                        echo "Container ${APP_NAME} does not exist."
                    }
                }
            }
        }                         
        stage('Deploy to container'){
            steps{
                sh 'docker run -d --name netflix -p 8081:80 ${DOCKER_USER}/${APP_NAME}:${IMAGE_TAG}'
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
            to: 'nd.onyima@gmail.com',                              
            attachmentsPattern: 'trivyfs.txt,trivyimage.txt'
        }
    }
}

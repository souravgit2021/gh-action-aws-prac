pipeline{
    agent any
    tools {
        maven 'MVN3.9'
        jdk 'JDK17'
    }

    environment {
        DOCKER_HUB_USER = 'docsourav1992'
        IMAGE_NAME      = 'spring_pet_clinic'
        IMAGE_TAG       = "${env.BUILD_NUMBER}"
        DOCKER_CRED_ID  = 'docker-hub-credentials' 
    }

    stages {

        stage("Code Testing"){
            steps{
                sh 'mvn -DskipTests test'
            }
        }

        stage("Sonarqube Analysis") {
            steps {
                script {
                    withSonarQubeEnv(credentialsId: 'sonar-token') {
                        sh "mvn sonar:sonar"
                    }
                }
            }

        }

        stage("App Code Build"){
            steps{
                sh 'mvn -DskipTests install'
            }
        }


        stage('Build Docker Image') {
            steps {
                script {
                    echo "Building image: ${DOCKER_HUB_USER}/${IMAGE_NAME}:${IMAGE_TAG}"
                    sh "docker build -t ${DOCKER_HUB_USER}/${IMAGE_NAME}:${IMAGE_TAG} ."
                    sh "docker tag ${DOCKER_HUB_USER}/${IMAGE_NAME}:${IMAGE_TAG} ${DOCKER_HUB_USER}/${IMAGE_NAME}:latest"
                }
            }
        }

        stage('Push to Docker Hub') {
            steps {
                withCredentials([usernamePassword(credentialsId: "${DOCKER_CRED_ID}", 
                                                 usernameVariable: 'DOCKER_USER', 
                                                 passwordVariable: 'DOCKER_PASS')]) {
                    script {
                        echo "Logging into Docker Hub..."
                        sh "echo \$DOCKER_PASS | docker login -u \$DOCKER_USER --password-stdin"
                        
                        echo "Pushing images..."
                        sh "docker push ${DOCKER_HUB_USER}/${IMAGE_NAME}:${IMAGE_TAG}"
                        sh "docker push ${DOCKER_HUB_USER}/${IMAGE_NAME}:latest"
                    }
                }
            }
        }

        stage('Removing The Local Docker Image'){
            steps{
                sh 'docker image rm ${DOCKER_HUB_USER}/${IMAGE_NAME}:${IMAGE_TAG}'
                sh 'docker image rm ${DOCKER_HUB_USER}/${IMAGE_NAME}:latest'

            }
        }



        }
    }
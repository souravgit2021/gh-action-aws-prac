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
                sh 'mvn -s settings.xml -Dcheckstyle.skip=true test'
            }
        }

        stage("Sonarqube Analysis") {
            steps {
                script {
                    withSonarQubeEnv(credentialsId: 'sonar-token') {
                        sh "mvn -Dcheckstyle.skip=true -s settings.xml sonar:sonar"
                    }
                }
            }

        }

        stage("App Code Build"){
            steps{
                sh 'mvn -s settings.xml -Dcheckstyle.skip=true install'
            }
        }


        stage('upload to nexus') {
            steps {
                sh 'pwd'
                nexusArtifactUploader artifacts: [[artifactId: 'spring-petclinic', classifier: '', file: 'spring-petclinic/target/spring-petclinic-4.0.0-SNAPSHOT.jar', type: '.jar']], credentialsId: 'nexus-login', groupId: 'org.springframework.samples', nexusUrl: '172.31.41.240:8081', nexusVersion: 'nexus3', protocol: 'http', repository: 'maven-snapshots', version: '4.0.0-SNAPSHOT'
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
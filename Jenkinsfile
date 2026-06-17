pipeline {
    agent any
    tools {
        maven 'MVN3.9'
        jdk 'JDK17'
    }

    environment {
        DOCKER_HUB_USER  = 'docsourav1992'
        IMAGE_NAME       = 'spring_pet_clinic'
        IMAGE_TAG        = "${env.BUILD_NUMBER}" 
        DOCKER_CRED_ID   = 'docker-hub-credentials' 
        GITHUB_CREDS     = credentials('git-login')
        REPO_URL         = 'github.com/souravgit2021/gitops-springpetclinic.git'
        BRANCH           = 'main'

        AWS_REGISTRY_ID  = '102512866166' 
        AWS_REGION       = 'us-east-2'     
        ECR_REPO_NAME    = 'spc-app'       
        
        ECR_REGISTRY_URI = "${AWS_REGISTRY_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
        IMAGE_URI        = "${ECR_REGISTRY_URI}/${ECR_REPO_NAME}"
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

        stage("Quality Gate") {
            steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'sonar-token'
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
                nexusArtifactUploader artifacts: [[artifactId: 'spring-petclinic', classifier: '', file: 'target/spring-petclinic-4.0.0-SNAPSHOT.jar', type: '.jar']], credentialsId: 'nexus-login', groupId: 'org.springframework.samples', nexusUrl: '172.31.41.240:8081', nexusVersion: 'nexus3', protocol: 'http', repository: 'maven-snapshots', version: '4.0.0-SNAPSHOT'
           }
        }

        stage('Build Docker Hub Image') { 
            steps {
                script {
                    echo "Building image: ${DOCKER_HUB_USER}/${IMAGE_NAME}:${IMAGE_TAG}"
                    sh "docker build -t ${DOCKER_HUB_USER}/${IMAGE_NAME}:${IMAGE_TAG} ."
                    sh "docker tag ${DOCKER_HUB_USER}/${IMAGE_NAME}:${IMAGE_TAG} ${DOCKER_HUB_USER}/${IMAGE_NAME}:latest"
                }
            }
        }

        stage('Trivy Security Scan') {
            steps {
                script {
                    sh "trivy image --severity HIGH,CRITICAL ${DOCKER_HUB_USER}/${IMAGE_NAME}:${IMAGE_TAG}"
                    sh "trivy image --format json --output trivy-report.json ${DOCKER_HUB_USER}/${IMAGE_NAME}:${IMAGE_TAG}"
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

        stage('Checkout Code For GitOps') {
            steps {
                git branch: "${BRANCH}", 
                    credentialsId: 'git-login', 
                    url: "https://${REPO_URL}"
            }
        }

        stage('Modify Deployment File') {
            steps {
                script {
                    sh "ls -l"
                    sh "sh "sed -i 's|^[[:space:]]*image:.*|        image: ${DOCKER_HUB_USER}/${IMAGE_NAME}:${IMAGE_TAG}|' deployment.yaml""
                }
            }
        }

        stage('Push Changes to GitHub') {
            steps {
                script {
                    sh "git config user.name 'Sourav Biswas'"
                    sh "git config user.email 'Sourav@mail.com'"
                    
                    sh "git add deployment.yaml"
                    sh "git commit -m 'automated deployment file update [skip ci]'"
                    
                    sh "git push https://${GITHUB_CREDS_USR}:${GITHUB_CREDS_PSW}@${REPO_URL} HEAD:${BRANCH}"
                }
            }
        }
        
        stage("Deploy To ECR And ECS"){
            // Corrected syntax for Declarative Pipeline input block
            input {
                message "Are We Good To Deploy To Production"
                ok "Proceed to Deploy"
            }
            steps {
                echo "Deployment approved by user."
            }
        }
        
        stage('Docker Login to AWS ECR') {
            steps { // Fixed formatting and misplaced bracket here
                script {
                    sh "aws ecr get-login-password --region us-east-2 | docker login --username AWS --password-stdin 102512866166.dkr.ecr.us-east-2.amazonaws.com"
                }
            }
        }

        stage('Build ECR Docker Image') { 
            steps {
                script {
                    sh "docker build -t ${ECR_REPO_NAME}:${IMAGE_TAG} ."
                }
            }
        }

         stage('Tag Docker Image') {
            steps {
                script {
                    sh "docker tag ${ECR_REPO_NAME}:${IMAGE_TAG} ${IMAGE_URI}:${IMAGE_TAG}"
                    sh "docker tag ${ECR_REPO_NAME}:${IMAGE_TAG} ${IMAGE_URI}:latest"
                }
            }
        }

        stage('Push Image to Amazon ECR') {
            steps {
                script {
                    sh "docker push ${IMAGE_URI}:${IMAGE_TAG}"
                    sh "docker push ${IMAGE_URI}:latest"
                }
            }
        }

        stage('Removing The Local Docker Image'){
            steps{
                sh 'docker image rm ${DOCKER_HUB_USER}/${IMAGE_NAME}:${IMAGE_TAG}'
                sh 'docker image rm ${DOCKER_HUB_USER}/${IMAGE_NAME}:latest'
                sh "docker rmi ${ECR_REPO_NAME}:${IMAGE_TAG} || true"
                sh "docker rmi ${IMAGE_URI}:${IMAGE_TAG} || true"
                sh "docker rmi ${IMAGE_URI}:latest || true"
            }
        }

        stage('Deploy to Amazon ECS') {
            environment {
                AWS_DEFAULT_REGION = 'us-east-2'
                ECS_CLUSTER_NAME   = 'superb-gecko-tukddh'
                ECS_SERVICE_NAME   = 'spc-jenkins-cicd-service-4iz15544'
            }
            steps {
                script {
                    sh """
                        aws ecs update-service \
                            --cluster ${ECS_CLUSTER_NAME} \
                            --service ${ECS_SERVICE_NAME} \
                            --force-new-deployment \
                            --region ${AWS_DEFAULT_REGION}
                    """
                }
            }
        }
    }

    post {
        always {
            cleanWs()
        }
    }
} // Fixed: Added final closing bracket for the pipeline

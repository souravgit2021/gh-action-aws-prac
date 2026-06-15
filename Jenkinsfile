pipeline {
    agent any
    tools {
        maven 'MVN3.9'
        jdk 'JDK17'
    }

    environment {
        DOCKER_HUB_USER  = 'docsourav1992'
        IMAGE_NAME       = 'spring_pet_clinic'
        IMAGE_TAG        = "${env.BUILD_NUMBER}" // Standardized single definition
        DOCKER_CRED_ID   = 'docker-hub-credentials' 
        GITHUB_CREDS     = credentials('git-login')
        REPO_URL         = 'github.com/souravgit2021/gitops-springpetclinic.git'
        BRANCH           = 'main'

        AWS_REGISTRY_ID  = '102512866166' // Your 12-digit AWS Account ID
        AWS_REGION       = 'us-east-2'     // Your target AWS Region
        ECR_REPO_NAME    = 'spc-app'       // Your Amazon ECR Repository Name
        
        // Formulated Variables
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

        stage('Build Docker Hub Image') { // Renamed to ensure uniqueness
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
                    // 1. Run a scan to print standard table output in the Jenkins console logs
                    sh "trivy image --severity HIGH,CRITICAL ${DOCKER_HUB_USER}/${IMAGE_NAME}:${IMAGE_TAG}"
                    
                    // 2. Optional: Generate a JSON report to archive as an artifact
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

        stage('Docker Login to AWS ECR') {
            steps 
               {
                    script {
                        sh "aws ecr get-login-password --region us-east-2 | docker login --username AWS --password-stdin 102512866166.dkr.ecr.us-east-2.amazonaws.com"
                    }
                }
            }
        

        stage('Build ECR Docker Image') { // Renamed to ensure uniqueness
            steps {
                script {
                    // Builds the image locally using the current workspace context
                    sh "docker build -t ${ECR_REPO_NAME}:${IMAGE_TAG} ."
                }
            }
        }

         stage('Tag Docker Image') {
            steps {
                script {
                    // Tags local build with ECR naming patterns
                    sh "docker tag ${ECR_REPO_NAME}:${IMAGE_TAG} ${IMAGE_URI}:${IMAGE_TAG}"
                    sh "docker tag ${ECR_REPO_NAME}:${IMAGE_TAG} ${IMAGE_URI}:latest"
                }
            }
        }

        stage('Push Image to Amazon ECR') {
            steps {
                script {
                    // Pushes both the build-versioned tag and the latest tag to ECR
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

        stage('Checkout Code') {
            steps {
                // Safely clones the repository using your credentials
                git branch: "${BRANCH}", 
                    credentialsId: 'git-login', 
                    url: "https://${REPO_URL}"
            }
        }

        stage('Modify Deployment File') {
            steps {
                script {
                    sh "ls -l"
                    sh "sed -i 's|^.*image:.*\$|      image: ${DOCKER_HUB_USER}/${IMAGE_NAME}:${IMAGE_TAG}|' deployment.yaml"
                }
            }
        }

        stage('Push Changes to GitHub') {
            steps {
                script {
                    // Configure temporary git identity for the commit
                    sh "git config user.name 'Sourav Biswas'"
                    sh "git config user.email 'Sourav@mail.com'"
                    
                    // Stage and commit the changed deployment file
                    sh "git add deployment.yaml"
                    sh "git commit -m 'automated deployment file update [skip ci]'"
                    
                    // Push back to GitHub using securely injected credentials
                    sh "git push https://${GITHUB_CREDS_USR}:${GITHUB_CREDS_PSW}@${REPO_URL} HEAD:${BRANCH}"
                }
            }
        }

    }

    post {
        always {
            // Archive the generated scan report inside Jenkins
            archiveArtifacts artifacts: 'trivy-report.json', allowEmptyArchive: true
        }
    }
}
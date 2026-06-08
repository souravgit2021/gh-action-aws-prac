pipeline {
    agent any
    environment {
        // Point Maven to a persistent, job-specific folder on the agent's drive
        MAVEN_OPTS = "-Dmaven.repo.local=${WORKSPACE}/.m2/repository"
    }
    stages {
        stage('Build') {
            steps {
                sh 'mvn clean install -Dcheckstyle.skip'
            }
        }

        stage("Verify Local Files"){
            steps{
                echo "Verifying local path contents..."
                sh 'ls -la target/'
            }
        }

        stage("Archiving Artifacts"){
            steps{
                archiveArtifacts artifacts: 'target/*.jar', fingerprint: true, allowEmptyArchive: false
            }
        }
    }
}
pipeline {
    agent any
    environment {
        // Point Maven to a persistent, job-specific folder on the agent's drive
        MAVEN_OPTS = "-Dmaven.repo.local=${WORKSPACE}/.m2/repository"
    }
    stages {
        stage('Build') {
            steps {
                sh 'mvn -DskipTests install'
            }
        }
    }
}
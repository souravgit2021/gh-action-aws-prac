pipeline{
    agent any
    tools {
        maven 'MVN3.9'
        jdk 'JDK17'
    }

    stages {

        stage("Code Testing"){
            steps{
                sh 'mvn -DskipTests test'
            }
        }

        }
    }
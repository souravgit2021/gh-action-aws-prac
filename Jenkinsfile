pipeline{
    agent any
    stages {
        // stage("Code CheckOut"){
        //     steps{
        //         git branch: 'main', credentialsId: 'none', url: 'https://github.com/souravgit2021/gh-action-aws-prac.git'
        //     }

        stage("Code Testing"){
            steps{
                mvn -DskipTests test
            }
        }

        }
    }
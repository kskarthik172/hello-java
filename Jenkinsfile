pipeline {
    agent any

    tools {
        maven 'maven-3.9'
        jdk 'jdk-11'
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonarqube-server') {
                    sh '''
                        mvn clean verify sonar:sonar \
                        -Dsonar.projectKey=hello-java \
                        -Dsonar.projectName=hello-java
                    '''
                }
            }
        }

        stage('Build & Generate Artifact') {
            steps {
                sh 'mvn clean package'
            }
        }
    }

    post {
        success {
            archiveArtifacts artifacts: 'target/*.jar'
            echo 'Pipeline executed successfully'
        }
    }
}



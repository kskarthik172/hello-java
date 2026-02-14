pipeline {
    agent any

    tools {
        maven 'maven-3.9'
        jdk 'jdk-11'
    }

    environment {
        SONAR_PROJECT_KEY = 'hello-java'

        // Nexus Docker Registry (NodePort)
        NEXUS_DOCKER_REGISTRY = '15.206.206.26:32082'
        IMAGE_NAME = 'hello-java'
        IMAGE_TAG = "${BUILD_NUMBER}"
    }

    stages {

        /* ---------- 1. CHECKOUT ---------- */
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        /* ---------- 2. SONARQUBE ---------- */
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonarqube-server') {
                    withCredentials([string(
                        credentialsId: 'sonar-token',
                        variable: 'SONAR_TOKEN'
                    )]) {
                        sh '''
                            mvn clean verify sonar:sonar \
                            -Dsonar.projectKey=${SONAR_PROJECT_KEY} \
                            -Dsonar.projectName=${SONAR_PROJECT_KEY} \
                            -Dsonar.login=$SONAR_TOKEN
                        '''
                    }
                }
            }
        }

        /* ---------- 3. BUILD ARTIFACT ---------- */
        stage('Build & Generate Artifact') {
            steps {
                sh 'mvn clean package -DskipTests'
                archiveArtifacts artifacts: 'target/*.jar'
            }
        }

        /* ---------- 4. STORE ARTIFACT IN NEXUS (MAVEN) ---------- */
        stage('Upload Artifact to Nexus (Maven)') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'nexus-creds',
                    usernameVariable: 'NEXUS_USER',
                    passwordVariable: 'NEXUS_PASS'
                )]) {
                    configFileProvider([
                        c


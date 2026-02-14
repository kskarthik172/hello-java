pipeline {
    agent any

    tools {
        maven 'maven-3.9'
        jdk 'jdk-11'
    }

    environment {
        // Sonar
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

        /* ---------- 2. SONARQUBE ANALYSIS ---------- */
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonarqube-server') {
                    withCredentials([string(
                        credentialsId: 'sonar-token',
                        variable: 'SONAR_TOKEN'
                    )]) {
                        sh """
                            mvn clean verify sonar:sonar \
                            -Dsonar.projectKey=${SONAR_PROJECT_KEY} \
                            -Dsonar.projectName=${SONAR_PROJECT_KEY} \
                            -Dsonar.login=${SONAR_TOKEN}
                        """
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

        /* ---------- 4. UPLOAD ARTIFACT TO NEXUS (MAVEN) ---------- */
        stage('Upload Artifact to Nexus (Maven)') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'nexus-creds',
                    usernameVariable: 'NEXUS_USER',
                    passwordVariable: 'NEXUS_PASS'
                )]) {
                    configFileProvider([
                        configFile(
                            fileId: 'maven-settings',
                            variable: 'MAVEN_SETTINGS'
                        )
                    ]) {
                        sh 'mvn deploy -DskipTests -s $MAVEN_SETTINGS'
                    }
                }
            }
        }

        /* ---------- 5. BUILD & PUSH DOCKER IMAGE ---------- */
        stage('Build & Push Docker Image') {
            steps {
                // Rebuild to ensure jar exists for Docker
                sh 'mvn clean package -DskipTests'

                withCredentials([usernamePassword(
                    credentialsId: 'nexus-creds',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh """
                        docker build -t ${NEXUS_DOCKER_REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG} .
                        echo ${DOCKER_PASS} | docker login ${NEXUS_DOCKER_REGISTRY} \
                          -u ${DOCKER_USER} --password-stdin
                        docker push ${NEXUS_DOCKER_REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}
                    """
                }
            }
        }
    }

    post {
        success {
            echo 'Pipeline executed successfully'
        }
        failure {
            echo 'Pipeline failed'
        }
    }
}


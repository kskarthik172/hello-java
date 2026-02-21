pipeline {
    agent any

    tools {
        maven 'maven-3.9'
        jdk 'jdk-11'
    }

    environment {
        SONAR_PROJECT_KEY = 'hello-java'
        NEXUS_DOCKER_REGISTRY = '15.206.206.26:32082'
        IMAGE_NAME = 'hello-java'
        IMAGE_TAG = "${BUILD_NUMBER}"

        AWS_REGION = 'ap-south-1'
        AWS_ACCOUNT_ID = '420838436623'
        ECR_REPO = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${IMAGE_NAME}"
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
                    withCredentials([string(
                        credentialsId: 'sonar-token',
                        variable: 'SONAR_TOKEN'
                    )]) {
                        sh '''
                            mvn clean verify sonar:sonar \
                              -Dsonar.projectKey=hello-java \
                              -Dsonar.projectName=hello-java \
                              -Dsonar.login=$SONAR_TOKEN
                        '''
                    }
                }
            }
        }

        stage('Build JAR') {
            steps {
                sh 'mvn clean package -DskipTests'
                archiveArtifacts artifacts: 'target/*.jar'
            }
        }

        stage('Upload JAR to Nexus') {
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

        stage('Build & Push Docker Image to Nexus') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'nexus-creds',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh '''
                        docker build -t ${NEXUS_DOCKER_REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG} .
                        echo "$DOCKER_PASS" | docker login ${NEXUS_DOCKER_REGISTRY} \
                          -u "$DOCKER_USER" --password-stdin
                        docker push ${NEXUS_DOCKER_REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}
                    '''
                }
            }
        }

        stage('Tag & Push Image to ECR') {
            steps {
                sh '''
                    echo "Logging into AWS ECR..."
                    aws ecr get-login-password --region ${AWS_REGION} | \
                    docker login --username AWS \
                    --password-stdin ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com

                    echo "Tagging image for ECR..."
                    docker tag ${NEXUS_DOCKER_REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG} \
                    ${ECR_REPO}:${IMAGE_TAG}

                    echo "Pushing image to ECR..."
                    docker push ${ECR_REPO}:${IMAGE_TAG}
                '''
            }
        }
    }

    post {
        success {
            echo "🎉 CI Pipeline completed successfully and image pushed to ECR"
        }
        failure {
            echo "❌ Pipeline failed"
        }
    }
}

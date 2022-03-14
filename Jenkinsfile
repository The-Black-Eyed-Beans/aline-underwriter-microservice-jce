pipeline {
    agent any

    tools {
        //Using the latest version of Maven
        maven "Maven 3.8.4"
    }
    
    environment {
        AWS_REGION                  = credentials('AWS_REGION')
        AWS_USER_ID                 = credentials('AWS_USER_ID')
        MICROSERVICE_IMAGE_NAME     = 'underwriter-microservice-jce'
        TAG                         = "${BUILD_NUMBER}_${BUILD_ID}"
    }

    stages {
        stage('Checkout') {
            steps {
                sh 'git submodule init'
                sh 'git submodule update'
            }
        }
        
        stage('Packaging'){
            steps{
                // Run Maven on a Unix agent.
                sh "mvn clean package -DskipTests"
            }
        }
        
        stage('Sonar Scan'){
            steps {
                withSonarQubeEnv(installationName: 'SonarQube-Server'){
                    sh "mvn verify -Dmaven.test.failure.ignore=true sonar:sonar -Dsonar.projectName=${MICROSERVICE_IMAGE_NAME}"
                }
            }
        }
        
        stage('Quality Gate'){
            steps {
                timeout(time: 3, unit: 'MINUTES'){
                    waitForQualityGate abortPipeline: true
                }
            }
        }
        
        stage('Build Image'){
            steps{
                sh "docker build -t ${MICROSERVICE_IMAGE_NAME}:${TAG} -f Underwriter-Dockerfile ."
            }
        }
        
        stage('Push Image'){
            steps{
                withAWS(credentials: 'jce-key') {
                    sh "aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${AWS_USER_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
                    // Push current build to ECR with build number and date-time
                    sh "docker tag ${MICROSERVICE_IMAGE_NAME}:${TAG} ${AWS_USER_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${MICROSERVICE_IMAGE_NAME}:${TAG}"
                    sh "docker push ${AWS_USER_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${MICROSERVICE_IMAGE_NAME}:${TAG}"
                    // Push current build to ECR as latest
                    sh "docker tag ${MICROSERVICE_IMAGE_NAME}:${TAG} ${AWS_USER_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${MICROSERVICE_IMAGE_NAME}:latest"
                    sh "docker push ${AWS_USER_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${MICROSERVICE_IMAGE_NAME}:latest"
                }
            }
        }
    }
    
    post {
        success{
            //Remove images locally
            sh "docker rmi ${MICROSERVICE_IMAGE_NAME}:${TAG}"
            sh "docker rmi ${AWS_USER_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${MICROSERVICE_IMAGE_NAME}:${TAG}"
            sh "docker rmi ${AWS_USER_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${MICROSERVICE_IMAGE_NAME}:latest"
        }
    }
}

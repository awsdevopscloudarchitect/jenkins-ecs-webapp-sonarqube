
pipeline {
    agent any

    options {
        timestamps()
    }

    environment {
        AWS_REGION = 'ap-south-1'
        AWS_ACCOUNT_ID = '466401735110'
        ECR_REPO = 'jenkins-ecs-app'
        ECS_CLUSTER = 'jenkins-ecs-cluster'
        ECS_SERVICE = 'jenkins-ecs-webapp-task'
        TASK_FAMILY = 'jenkins-ecs-webapp-task'
        CONTAINER_NAME = 'webapp'
    }

    stages {
        stage('Git Checkout') {
            steps {
                checkout scm
            }
        }

        stage('SonarQube Analysis') {
            steps {
                script {
                    def scannerHome = tool 'SonarScanner'

                    withSonarQubeEnv('SonarQube') {
                        sh """
                            ${scannerHome}/bin/sonar-scanner \
                            -Dsonar.projectKey=jenkins-ecs-webapp \
                            -Dsonar.sources=. \
                            -Dsonar.inclusions=index.html
                        """
                    }
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                sh 'docker build -t $ECR_REPO:$BUILD_NUMBER .'
            }
        }

        stage('Trivy Security Scan') {
            steps {
                sh '''
                    trivy image --exit-code 0 \
                    --severity HIGH,CRITICAL \
                    $ECR_REPO:$BUILD_NUMBER
                '''
            }
        }

        stage('Login to ECR') {
            steps {
                sh '''
                    aws ecr get-login-password --region $AWS_REGION |
                    docker login --username AWS --password-stdin \
                    $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com
                '''
            }
        }

        stage('Push Image to ECR') {
            steps {
                sh '''
                    IMAGE_URI=$AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/$ECR_REPO:$BUILD_NUMBER
                    docker tag $ECR_REPO:$BUILD_NUMBER $IMAGE_URI
                    docker push $IMAGE_URI
                '''
            }
        }

        stage('Deploy to ECS') {
            steps {
                sh '''
                    export IMAGE_URI=$AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/$ECR_REPO:$BUILD_NUMBER

                    aws ecs describe-task-definition \
                    --task-definition $TASK_FAMILY \
                    --region $AWS_REGION \
                    --query taskDefinition > taskdef.json

                    jq --arg image "$IMAGE_URI" --arg name "$CONTAINER_NAME" '
                    del(
                        .taskDefinitionArn,
                        .revision,
                        .status,
                        .requiresAttributes,
                        .compatibilities,
                        .registeredAt,
                        .registeredBy,
                        .deregisteredAt
                    ) |
                    .containerDefinitions |= map(
                        if .name == $name then .image = $image else . end
                    )' taskdef.json > new-taskdef.json

                    TASK_DEF_ARN=$(aws ecs register-task-definition \
                    --cli-input-json file://new-taskdef.json \
                    --region $AWS_REGION \
                    --query taskDefinition.taskDefinitionArn \
                    --output text)

                    aws ecs update-service \
                    --cluster $ECS_CLUSTER \
                    --service $ECS_SERVICE \
                    --task-definition $TASK_DEF_ARN \
                    --region $AWS_REGION

                    aws ecs wait services-stable \
                    --cluster $ECS_CLUSTER \
                    --services $ECS_SERVICE \
                    --region $AWS_REGION
                '''
            }
        }
    }

    post {
        success {
            echo 'Application deployed successfully to ECS!'
        }
        failure {
            echo 'Pipeline failed. Check Jenkins Console Output.'
        }
    }
}

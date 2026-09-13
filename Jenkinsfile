pipeline {
    agent any

    environment {
        AWS_REGION     = 'us-east-1'
        AWS_ACCOUNT_ID = '434504868934'

        ECR_REPOSITORY = 'java-docker-app'
        ECR_REGISTRY   = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
        IMAGE_NAME     = "${ECR_REGISTRY}/${ECR_REPOSITORY}"

        ECS_CLUSTER    = 'java-app-cluster'
        ECS_SERVICE    = 'java-app-service'
        TASK_FAMILY    = 'java-app-task'
        CONTAINER_NAME = 'java-app'

        IMAGE_TAG      = "${BUILD_NUMBER}"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Maven Build') {
            steps {
                sh '''
                    echo "=== Maven Build ==="
                    mvn clean package -DskipTests
                '''
            }
        }

        stage('Maven Test') {
            steps {
                sh '''
                    echo "=== Maven Test ==="
                    mvn test
                '''
            }
        }

        stage('Docker Build') {
            steps {
                sh '''
                    echo "=== Docker Build ==="

                    docker build \
                      -t ${IMAGE_NAME}:${IMAGE_TAG} \
                      .

                    docker tag \
                      ${IMAGE_NAME}:${IMAGE_TAG} \
                      ${IMAGE_NAME}:latest
                '''
            }
        }

        stage('Login to ECR') {
            steps {
                sh '''
                    echo "=== Login to Amazon ECR ==="

                    aws ecr get-login-password \
                      --region ${AWS_REGION} | \
                    docker login \
                      --username AWS \
                      --password-stdin ${ECR_REGISTRY}
                '''
            }
        }

        stage('Push Image to ECR') {
            steps {
                sh '''
                    echo "=== Push Image to ECR ==="

                    docker push ${IMAGE_NAME}:${IMAGE_TAG}
                    docker push ${IMAGE_NAME}:latest
                '''
            }
        }

        stage('Create ECS Task Definition') {
            steps {
                sh '''
                    echo "=== Download Current ECS Task Definition ==="

                    aws ecs describe-task-definition \
                      --task-definition ${TASK_FAMILY} \
                      --region ${AWS_REGION} \
                      --query 'taskDefinition' \
                      --output json > task-definition.json

                    echo "=== Update Container Image ==="

                    python3 <<'PY'
import json
import os

with open("task-definition.json") as f:
    task = json.load(f)

image = os.environ["IMAGE_NAME"] + ":" + os.environ["IMAGE_TAG"]
container_name = os.environ["CONTAINER_NAME"]

for container in task["containerDefinitions"]:
    if container["name"] == container_name:
        container["image"] = image

remove_fields = [
    "taskDefinitionArn",
    "revision",
    "status",
    "requiresAttributes",
    "compatibilities",
    "registeredAt",
    "registeredBy"
]

for field in remove_fields:
    task.pop(field, None)

with open("new-task-definition.json", "w") as f:
    json.dump(task, f)

print("New image:", image)
PY

                    echo "=== Register New ECS Task Definition ==="

                    NEW_TASK_DEF_ARN=$(aws ecs register-task-definition \
                      --cli-input-json file://new-task-definition.json \
                      --region ${AWS_REGION} \
                      --query 'taskDefinition.taskDefinitionArn' \
                      --output text)

                    echo "New Task Definition:"
                    echo ${NEW_TASK_DEF_ARN}

                    echo ${NEW_TASK_DEF_ARN} > new-task-definition-arn.txt
                '''
            }
        }

        stage('Deploy to ECS') {
            steps {
                sh '''
                    echo "=== Deploy to ECS ==="

                    NEW_TASK_DEF_ARN=$(cat new-task-definition-arn.txt)

                    aws ecs update-service \
                      --cluster ${ECS_CLUSTER} \
                      --service ${ECS_SERVICE} \
                      --task-definition ${NEW_TASK_DEF_ARN} \
                      --force-new-deployment \
                      --region ${AWS_REGION}

                    echo "ECS deployment started."
                '''
            }
        }

        stage('Verify ECS Deployment') {
            steps {
                sh '''
                    echo "=== Waiting for ECS Service Stability ==="

                    aws ecs wait services-stable \
                      --cluster ${ECS_CLUSTER} \
                      --services ${ECS_SERVICE} \
                      --region ${AWS_REGION}

                    echo "=== ECS Service Status ==="

                    aws ecs describe-services \
                      --cluster ${ECS_CLUSTER} \
                      --services ${ECS_SERVICE} \
                      --region ${AWS_REGION} \
                      --query 'services[0].[serviceName,status,desiredCount,runningCount,pendingCount]' \
                      --output table

                    echo "=== Running Container Image ==="

                    TASK_ARN=$(aws ecs list-tasks \
                      --cluster ${ECS_CLUSTER} \
                      --service-name ${ECS_SERVICE} \
                      --desired-status RUNNING \
                      --region ${AWS_REGION} \
                      --query 'taskArns[0]' \
                      --output text)

                    aws ecs describe-tasks \
                      --cluster ${ECS_CLUSTER} \
                      --tasks ${TASK_ARN} \
                      --region ${AWS_REGION} \
                      --query 'tasks[0].containers[?name==`java-app`].image' \
                      --output text
                '''
            }
        }
    }

    post {
        success {
            echo '========================================'
            echo 'PROJECT 5 DEPLOYMENT SUCCESSFUL'
            echo '========================================'
            echo "Docker Image: ${IMAGE_NAME}:${IMAGE_TAG}"
            echo "ECS Service: ${ECS_SERVICE}"
        }

        failure {
            echo '========================================'
            echo 'PROJECT 5 PIPELINE FAILED'
            echo '========================================'
            echo 'Deployment stopped because a pipeline stage failed.'
        }

        always {
            sh '''
                rm -f task-definition.json
                rm -f new-task-definition.json
                rm -f new-task-definition-arn.txt
            '''
        }
    }
}

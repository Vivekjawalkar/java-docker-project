pipeline {
    agent any

    environment {
        JAVA_HOME = '/usr/lib/jvm/java-21-amazon-corretto.x86_64'
        PATH = "${JAVA_HOME}/bin:${env.PATH}"

        AWS_REGION     = 'us-east-1'
        AWS_ACCOUNT_ID = '434504868934'

        ECR_REPOSITORY = 'java-docker-app'
        ECR_REGISTRY   = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
        IMAGE_NAME     = "${ECR_REGISTRY}/${ECR_REPOSITORY}"

        ECS_CLUSTER = 'java-app-cluster'

        DEV_SERVICE      = 'java-app-dev-service'
        DEV_TASK_FAMILY  = 'java-app-dev-task'

        PROD_SERVICE     = 'java-app-prod-service'
        PROD_TASK_FAMILY = 'java-app-prod-task'

        CONTAINER_NAME = 'java-app'

        IMAGE_TAG = "${BUILD_NUMBER}"
    }

    stages {

        stage('Checkout') {
            steps {
                echo '=== Checkout Source Code ==='
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

                    echo "Docker image created:"
                    echo "${IMAGE_NAME}:${IMAGE_TAG}"
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

                    echo "Image pushed successfully:"
                    echo "${IMAGE_NAME}:${IMAGE_TAG}"
                '''
            }
        }

        stage('Deploy to DEV') {
            steps {
                sh '''
                    echo "=== DEV DEPLOYMENT ==="

                    aws ecs describe-task-definition \
                      --task-definition ${DEV_TASK_FAMILY} \
                      --region ${AWS_REGION} \
                      --query 'taskDefinition' \
                      --output json > dev-task-definition.json

                    python3 <<'PY'
import json
import os

with open("dev-task-definition.json") as f:
    task = json.load(f)

image = os.environ["IMAGE_NAME"] + ":" + os.environ["IMAGE_TAG"]
container_name = os.environ["CONTAINER_NAME"]

for container in task["containerDefinitions"]:
    if container["name"] == container_name:
        container["image"] = image

for field in [
    "taskDefinitionArn",
    "revision",
    "status",
    "requiresAttributes",
    "compatibilities",
    "registeredAt",
    "registeredBy"
]:
    task.pop(field, None)

with open("dev-task-definition-new.json", "w") as f:
    json.dump(task, f)

print("DEV image:", image)
PY

                    DEV_TASK_DEF_ARN=$(aws ecs register-task-definition \
                      --cli-input-json file://dev-task-definition-new.json \
                      --region ${AWS_REGION} \
                      --query 'taskDefinition.taskDefinitionArn' \
                      --output text)

                    echo "DEV Task Definition:"
                    echo "${DEV_TASK_DEF_ARN}"

                    aws ecs update-service \
                      --cluster ${ECS_CLUSTER} \
                      --service ${DEV_SERVICE} \
                      --task-definition ${DEV_TASK_DEF_ARN} \
                      --force-new-deployment \
                      --region ${AWS_REGION}

                    echo "DEV deployment started."
                '''
            }
        }

        stage('Verify DEV') {
            steps {
                sh '''
                    echo "=== VERIFY DEV ==="

                    aws ecs wait services-stable \
                      --cluster ${ECS_CLUSTER} \
                      --services ${DEV_SERVICE} \
                      --region ${AWS_REGION}

                    echo "=== DEV SERVICE STATUS ==="

                    aws ecs describe-services \
                      --cluster ${ECS_CLUSTER} \
                      --services ${DEV_SERVICE} \
                      --region ${AWS_REGION} \
                      --query 'services[0].[serviceName,status,desiredCount,runningCount,pendingCount,taskDefinition]' \
                      --output table

                    echo "=== DEV RUNNING IMAGE ==="

                    TASK_ARN=$(aws ecs list-tasks \
                      --cluster ${ECS_CLUSTER} \
                      --service-name ${DEV_SERVICE} \
                      --desired-status RUNNING \
                      --region ${AWS_REGION} \
                      --query 'taskArns[0]' \
                      --output text)

                    DEV_IMAGE=$(aws ecs describe-tasks \
                      --cluster ${ECS_CLUSTER} \
                      --tasks ${TASK_ARN} \
                      --region ${AWS_REGION} \
                      --query 'tasks[0].containers[?name==`java-app`].image' \
                      --output text)

                    echo "DEV Running Image:"
                    echo "${DEV_IMAGE}"

                    EXPECTED_IMAGE="${IMAGE_NAME}:${IMAGE_TAG}"

                    if [ "${DEV_IMAGE}" != "${EXPECTED_IMAGE}" ]; then
                        echo "ERROR: DEV image does not match expected image."
                        exit 1
                    fi

                    echo "DEV image verification successful."
                '''
            }
        }

        stage('Manual Approval for PROD') {
            steps {
                timeout(time: 10, unit: 'MINUTES') {
                    input message: 'DEV deployment verified. Deploy this image to PROD?', \
                          ok: 'Deploy to PROD'
                }
            }
        }

        stage('Deploy to PROD') {
            steps {
                sh '''
                    echo "=== PROD DEPLOYMENT ==="

                    aws ecs describe-task-definition \
                      --task-definition ${PROD_TASK_FAMILY} \
                      --region ${AWS_REGION} \
                      --query 'taskDefinition' \
                      --output json > prod-task-definition.json

                    python3 <<'PY'
import json
import os

with open("prod-task-definition.json") as f:
    task = json.load(f)

image = os.environ["IMAGE_NAME"] + ":" + os.environ["IMAGE_TAG"]
container_name = os.environ["CONTAINER_NAME"]

for container in task["containerDefinitions"]:
    if container["name"] == container_name:
        container["image"] = image

for field in [
    "taskDefinitionArn",
    "revision",
    "status",
    "requiresAttributes",
    "compatibilities",
    "registeredAt",
    "registeredBy"
]:
    task.pop(field, None)

with open("prod-task-definition-new.json", "w") as f:
    json.dump(task, f)

print("PROD image:", image)
PY

                    PROD_TASK_DEF_ARN=$(aws ecs register-task-definition \
                      --cli-input-json file://prod-task-definition-new.json \
                      --region ${AWS_REGION} \
                      --query 'taskDefinition.taskDefinitionArn' \
                      --output text)

                    echo "PROD Task Definition:"
                    echo "${PROD_TASK_DEF_ARN}"

                    aws ecs update-service \
                      --cluster ${ECS_CLUSTER} \
                      --service ${PROD_SERVICE} \
                      --task-definition ${PROD_TASK_DEF_ARN} \
                      --force-new-deployment \
                      --region ${AWS_REGION}

                    echo "PROD deployment started."
                '''
            }
        }

        stage('Verify PROD') {
            steps {
                sh '''
                    echo "=== VERIFY PROD ==="

                    aws ecs wait services-stable \
                      --cluster ${ECS_CLUSTER} \
                      --services ${PROD_SERVICE} \
                      --region ${AWS_REGION}

                    echo "=== PROD SERVICE STATUS ==="

                    aws ecs describe-services \
                      --cluster ${ECS_CLUSTER} \
                      --services ${PROD_SERVICE} \
                      --region ${AWS_REGION} \
                      --query 'services[0].[serviceName,status,desiredCount,runningCount,pendingCount,taskDefinition]' \
                      --output table

                    echo "=== PROD RUNNING IMAGE ==="

                    TASK_ARN=$(aws ecs list-tasks \
                      --cluster ${ECS_CLUSTER} \
                      --service-name ${PROD_SERVICE} \
                      --desired-status RUNNING \
                      --region ${AWS_REGION} \
                      --query 'taskArns[0]' \
                      --output text)

                    PROD_IMAGE=$(aws ecs describe-tasks \
                      --cluster ${ECS_CLUSTER} \
                      --tasks ${TASK_ARN} \
                      --region ${AWS_REGION} \
                      --query 'tasks[0].containers[?name==`java-app`].image' \
                      --output text)

                    echo "PROD Running Image:"
                    echo "${PROD_IMAGE}"

                    EXPECTED_IMAGE="${IMAGE_NAME}:${IMAGE_TAG}"

                    if [ "${PROD_IMAGE}" != "${EXPECTED_IMAGE}" ]; then
                        echo "ERROR: PROD image does not match expected image."
                        exit 1
                    fi

                    echo "PROD image verification successful."
                '''
            }
        }
    }

    post {
        success {
            echo '========================================'
            echo 'PROJECT 7 DEV TO PROD SUCCESSFUL'
            echo '========================================'
            echo "Docker Image: ${IMAGE_NAME}:${IMAGE_TAG}"
            echo "DEV Service: ${DEV_SERVICE}"
            echo "PROD Service: ${PROD_SERVICE}"
        }

        failure {
            echo '========================================'
            echo 'PROJECT 7 PIPELINE FAILED'
            echo '========================================'
            echo 'Deployment stopped because a stage failed.'
        }

        always {
            sh '''
                rm -f dev-task-definition.json
                rm -f dev-task-definition-new.json
                rm -f prod-task-definition.json
                rm -f prod-task-definition-new.json
            '''
        }
    }
}

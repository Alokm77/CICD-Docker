pipeline {
    agent any
    tools {
        nodejs 'NodeJS' // Ensure NodeJS is configured in Jenkins
    }
    environment {
        SONAR_PROJECT_KEY = 'CICD-docker'
        SONAR_SCANNER_HOME = tool 'SonarQubeScanner'
        DOCKER_HUB_REPO = 'sampatel8543/cicd-project'
        JOB_NAME_NOW = 'cicd01'
        ECR_REPO = 'dev-repo'
        IMAGE_TAG = 'latest'
        ECR_REGISTRY = '905417999377.dkr.ecr.us-east-1.amazonaws.com'
        AWS_REGION = 'us-east-1'
        ECS_CLUSTER = 'dev-cluster'
        TASK_DEFINITION_NAME = 'dev-task'
        CONTAINER_NAME = 'dev-container'
        CONTAINER_PORT_1 = '80'
        CONTAINER_PORT_2 = '3000'
        ECS_SERVICE_NAME = 'dev-service'
        SUBNET_IDS = 'subnet-01c1111d3cee3fb0d,subnet-04030489d75ed52ec' // Replace with your subnet IDs
        ALB_NAME = 'dev-alb'
        TARGET_GROUP_NAME = 'dev-target-group'
        VPC_ID = 'vpc-0bd5e05b7eb883a10'  // Replace with your actual VPC ID
        IMAGE_URL = "httpd:2.4"
        CPU = "256"
        MEMORY = "512"
    }
    stages {
        stage('Create ECR Repository') {
            steps {
                script {
                    echo "Checking if ECR repository exists or creating it..."
                    sh """
                    aws ecr describe-repositories --repository-names ${ECR_REPO} --region ${AWS_REGION} || \
                    aws ecr create-repository --repository-name ${ECR_REPO} --region ${AWS_REGION}
                    """
                }
            }
        }

        stage('Create ECS Cluster') {
            steps {
                script {
                    echo "Checking if ECS cluster exists or creating it..."
                    sh """
                    aws ecs describe-clusters --clusters ${ECS_CLUSTER} --region ${AWS_REGION} --query "clusters[?status=='ACTIVE'].clusterName" --output text || \
                    aws ecs create-cluster --cluster-name ${ECS_CLUSTER} --region ${AWS_REGION}
                    """
                }
            }
        }

        stage('Create Security Group') {
            steps {
                script {
                    echo "Checking if Security Group exists..."
                    def securityGroupId = sh(
                        script: '''
                        aws ec2 describe-security-groups \
                            --filters Name=group-name,Values=dev-sg Name=vpc-id,Values=${VPC_ID} \
                            --query "SecurityGroups[0].GroupId" --output text
                        ''',
                        returnStdout: true
                    ).trim()

                    if (securityGroupId == "None") {
                        echo "Security Group not found. Creating a new Security Group..."
                        def newSecurityGroup = sh(
                            script: '''
                            aws ec2 create-security-group \
                                --group-name dev-sg \
                                --description "Dev security group" \
                                --vpc-id ${VPC_ID} \
                                --output json
                            ''',
                            returnStdout: true
                        ).trim()

                        // Use jq to parse the JSON output
                        securityGroupId = sh(
                            script: """
                            echo '${newSecurityGroup}' | jq -r '.GroupId'
                            """,
                            returnStdout: true
                        ).trim()
                        echo "Created new Security Group with ID: ${securityGroupId}"
                    } else {
                        echo "Found existing Security Group with ID: ${securityGroupId}"
                    }

                    // Store the security group ID in the environment
                    env.SECURITY_GROUP_ID = securityGroupId

                    echo "Checking for existing ingress rule..."
                    try {
                        sh """
                        aws ec2 authorize-security-group-ingress \
                            --group-id ${securityGroupId} \
                            --protocol tcp --port 80 --cidr 0.0.0.0/0 --region ${AWS_REGION}
                        """
                        echo "Ingress rule added successfully."
                    } catch (Exception e) {
                        echo "Ingress rule for port 80 already exists, skipping rule creation."
                    }
                }
            }
        }

        stage('Create Application Load Balancer') {
            steps {
                script {
                    echo "Creating an Application Load Balancer..."
                    def subnetIds = SUBNET_IDS.split(',')
                    sh """
                    aws elbv2 create-load-balancer \
                        --name ${ALB_NAME} \
                        --subnets ${subnetIds.join(' ')} \
                        --security-groups ${SECURITY_GROUP_ID} \
                        --type application \
                        --scheme internet-facing \
                        --region ${AWS_REGION} \
                        --query LoadBalancers[0].LoadBalancerArn \
                        --output text
                    """
                }
            }
        }

        stage('Create Target Group') {
            steps {
                script {
                    echo "Creating Target Group for Port ${CONTAINER_PORT_1}..."
                    def targetGroupArn = sh(
                        script: """
                        aws elbv2 create-target-group --name ${TARGET_GROUP_NAME} \
                            --protocol HTTP --port ${CONTAINER_PORT_1} \
                            --vpc-id ${VPC_ID} --target-type ip \
                            --region ${AWS_REGION} \
                            --query 'TargetGroups[0].TargetGroupArn' --output text
                        """,
                        returnStdout: true
                    ).trim()

                    echo "Target Group Created for Port ${CONTAINER_PORT_1}: ${targetGroupArn}"
                    env.TARGET_GROUP_ARN = targetGroupArn

                    echo "Creating Target Group for Port ${CONTAINER_PORT_2}..."
                    def targetGroupArn3000 = sh(
                        script: """
                        aws elbv2 create-target-group --name ${TARGET_GROUP_NAME}-3000 \
                            --protocol HTTP --port ${CONTAINER_PORT_2} \
                            --vpc-id ${VPC_ID} --target-type ip \
                            --region ${AWS_REGION} \
                            --query 'TargetGroups[0].TargetGroupArn' --output text
                        """,
                        returnStdout: true
                    ).trim()

                    echo "Target Group Created for Port ${CONTAINER_PORT_2}: ${targetGroupArn3000}"
                    env.TARGET_GROUP_ARN_3000 = targetGroupArn3000
                }
            }
        }

        stage('Create ECS Task Definition') {
            steps {
                script {
                    echo "Creating ECS Task Definition..."
                    try {
                        // Write the task definition JSON to a file
                        sh '''
                        cat > task-definition.json <<'EOF'
                        {
                            "family": "${TASK_DEFINITION_NAME}",
                            "containerDefinitions": [
                                {
                                    "name": "${CONTAINER_NAME}",
                                    "image": "${IMAGE_URL}",
                                    "memory": ${MEMORY},
                                    "cpu": ${CPU},
                                    "executionRoleArn": "arn:aws:iam::905417999377:role/aws-service-role/ecs.amazonaws.com/AWSServiceRoleForECS",
                                    "essential": true,
                                    "portMappings": [
                                        {
                                            "containerPort": ${CONTAINER_PORT_1},
                                            "hostPort": ${CONTAINER_PORT_1},
                                            "protocol": "tcp"
                                        },
                                        {
                                            "containerPort": ${CONTAINER_PORT_2},
                                            "hostPort": ${CONTAINER_PORT_2},
                                            "protocol": "tcp"
                                        }
                                    ]
                                }
                            ],
                            "networkMode": "awsvpc",
                            "requiresCompatibilities": ["FARGATE"],
                            "cpu": "${CPU}",
                            "memory": "${MEMORY}"
                        }
                        EOF
                        '''
                        // Verify the file content
                        sh "cat task-definition.json"
                        // Register the task definition
                        sh """
                        aws ecs register-task-definition --cli-input-json file://task-definition.json --region ${AWS_REGION}
                        """
                    } catch (Exception e) {
                        echo "Failed to create task definition: ${e}"
                        throw e
                    }
                }
            }
        }

        stage('Create ECS Service') {
            steps {
                script {
                    echo "Creating ECS Service..."
                    sh """
                    aws ecs create-service \
                        --cluster ${ECS_CLUSTER} \
                        --service-name ${ECS_SERVICE_NAME} \
                        --task-definition ${TASK_DEFINITION_NAME} \
                        --desired-count 1 \
                        --launch-type FARGATE \
                        --load-balancers "targetGroupArn=${TARGET_GROUP_ARN},containerName=${CONTAINER_NAME},containerPort=${CONTAINER_PORT_1}" \
                        --load-balancers "targetGroupArn=${TARGET_GROUP_ARN_3000},containerName=${CONTAINER_NAME},containerPort=${CONTAINER_PORT_2}" \
                        --network-configuration "awsvpcConfiguration={subnets=[${SUBNET_IDS}],securityGroups=[${SECURITY_GROUP_ID}],assignPublicIp=ENABLED}" \
                        --region ${AWS_REGION}
                    """
                }
            }
        }
    }
}

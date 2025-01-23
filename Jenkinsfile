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
                    aws ecs describe-clusters --clusters ${ECS_CLUSTER} --region ${AWS_REGION} --query 'clusters[?status==`ACTIVE`].clusterName' --output text || \
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
                    --filters Name=group-name,Values=dev-sg Name=vpc-id,Values=vpc-0bd5e05b7eb883a10 \
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
                        --vpc-id vpc-0bd5e05b7eb883a10 \
                        --output json
                    ''',
                    returnStdout: true
                ).trim()
                
                // Extract GroupId from the JSON response
                def jsonResponse = readJSON text: newSecurityGroup
                securityGroupId = jsonResponse.GroupId
                echo "Created new Security Group with ID: ${securityGroupId}"
            } else {
                echo "Found existing Security Group with ID: ${securityGroupId}"
            }

            echo "Checking for existing ingress rule..."
            def ruleExists = sh(
                script: """
                aws ec2 describe-security-groups \
                    --group-ids ${securityGroupId} \
                    --query "SecurityGroups[0].IpPermissions[?FromPort=='80' && ToPort=='80' && IpProtocol=='tcp' && contains(IpRanges[].CidrIp, '0.0.0.0/0')]" \
                    --output text
                """,
                returnStdout: true
            ).trim()

            if (ruleExists) {
                echo "Ingress rule for port 80 already exists. Skipping rule creation."
            } else {
                echo "Authorizing ingress rules for Security Group..."
                sh """
                aws ec2 authorize-security-group-ingress \
                    --group-id ${securityGroupId} \
                    --protocol tcp --port 80 --cidr 0.0.0.0/0 --region us-east-1
                """
                echo "Ingress rule added successfully."
            }

            // Export the Security Group ID for subsequent stages
            env.SECURITY_GROUP_ID = securityGroupId
        }
    }
}
        stage('Create Application Load Balancer') {
            steps {
                script {
                    echo "Creating an Application Load Balancer..."
                    if (!env.SECURITY_GROUP_ID) {
                        error "SECURITY_GROUP_ID is not set. Ensure the Security Group is created successfully in the previous stage."
                    }

                    def loadBalancerArn = sh(
                        script: """
                        aws elbv2 create-load-balancer --name ${ALB_NAME} \
                            --subnets ${SUBNET_IDS} \
                            --security-groups ${SECURITY_GROUP_ID} \
                            --type application --scheme internet-facing \
                            --region ${AWS_REGION} \
                            --query LoadBalancers[0].LoadBalancerArn --output text
                        """,
                        returnStdout: true
                    ).trim()

                    echo "Application Load Balancer created with ARN: ${loadBalancerArn}"
                    env.LOAD_BALANCER_ARN = loadBalancerArn
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
                            --vpc-id vpc-xxxxxxx --target-type ip \
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
                            --vpc-id vpc-xxxxxxx --target-type ip \
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
                    sh """
                    cat > task-definition.json <<EOF
                    {
                        "family": "${TASK_DEFINITION_NAME}",
                        "containerDefinitions": [
                            {
                                "name": "${CONTAINER_NAME}",
                                "image": "${ECR_REGISTRY}/${ECR_REPO}:${IMAGE_TAG}",
                                "memory": 512,
                                "cpu": 256,
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
                        "cpu": "256",
                        "memory": "512"
                    }
                    EOF
                    aws ecs register-task-definition --cli-input-json file://task-definition.json --region ${AWS_REGION}
                    """
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

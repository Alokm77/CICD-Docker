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
        SECURITY_GROUP = '' // Will be created dynamically
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
                aws ec2 describe-security-groups --filters Name=group-name,Values=dev-sg Name=vpc-id,Values=vpc-0bd5e05b7eb883a10 --query "SecurityGroups[0].GroupId" --output text || echo "None"
                ''',
                returnStdout: true
            ).trim()
            
            if (securityGroupId == "None") {
                echo "Security Group not found. Creating a new one..."
                securityGroupId = sh(
                    script: '''
                    aws ec2 create-security-group --group-name dev-sg --description "Security group for ECS service" --vpc-id vpc-0bd5e05b7eb883a10 --query GroupId --output text
                    ''',
                    returnStdout: true
                ).trim()
                echo "Created Security Group with ID: ${securityGroupId}"
            } else {
                echo "Security Group already exists with ID: ${securityGroupId}"
            }

            // Export the security group ID for later use
            env.SECURITY_GROUP_ID = securityGroupId
            echo "Security Group ID: ${env.SECURITY_GROUP_ID}"

            // Authorize ingress rules
            echo "Authorizing ingress rules for Security Group..."
            sh '''
            aws ec2 authorize-security-group-ingress --group-id ${SECURITY_GROUP_ID} --protocol tcp --port 80 --cidr 0.0.0.0/0 --region us-east-1
            '''
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
                script: '''
                aws elbv2 create-load-balancer --name dev-alb \
                    --subnets subnet-01c1111d3cee3fb0d subnet-04030489d75ed52ec \
                    --security-groups ${SECURITY_GROUP_ID} \
                    --type application --scheme internet-facing \
                    --region us-east-1 \
                    --query LoadBalancers[0].LoadBalancerArn --output text
                ''',
                returnStdout: true
            ).trim()
            echo "Application Load Balancer created with ARN: ${loadBalancerArn}"
        }
    }
}
        stage('Create Target Group') {
            steps {
                script {
                    echo "Creating a Target Group..."
                    sh """
                    TARGET_GROUP_ARN=\$(aws elbv2 create-target-group --name ${TARGET_GROUP_NAME} --protocol HTTP --port ${CONTAINER_PORT_1} --vpc-id vpc-xxxxxxx --target-type ip --region ${AWS_REGION} --query 'TargetGroups[0].TargetGroupArn' --output text)
                    echo "Target Group Created for Port ${CONTAINER_PORT_1}: \$TARGET_GROUP_ARN"
                    echo "export TARGET_GROUP_ARN=\$TARGET_GROUP_ARN" >> env.properties
                    """
                    echo "Creating a Target Group for Port 3000..."
                    sh """
                    TARGET_GROUP_ARN_3000=\$(aws elbv2 create-target-group --name ${TARGET_GROUP_NAME}-3000 --protocol HTTP --port ${CONTAINER_PORT_2} --vpc-id vpc-xxxxxxx --target-type ip --region ${AWS_REGION} --query 'TargetGroups[0].TargetGroupArn' --output text)
                    echo "Target Group Created for Port ${CONTAINER_PORT_2}: \$TARGET_GROUP_ARN_3000"
                    echo "export TARGET_GROUP_ARN_3000=\$TARGET_GROUP_ARN_3000" >> env.properties
                    """
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
                        --load-balancers "targetGroupArn=\$(cat env.properties | grep TARGET_GROUP_ARN | cut -d= -f2),containerName=${CONTAINER_NAME},containerPort=${CONTAINER_PORT_1}" \
                        --load-balancers "targetGroupArn=\$(cat env.properties | grep TARGET_GROUP_ARN_3000 | cut -d= -f2),containerName=${CONTAINER_NAME},containerPort=${CONTAINER_PORT_2}" \
                        --network-configuration "awsvpcConfiguration={subnets=[${SUBNET_IDS}],securityGroups=[\$(cat env.properties | grep SECURITY_GROUP | cut -d= -f2)],assignPublicIp=ENABLED}" \
                        --region ${AWS_REGION}
                    """
                }
            }
        }
        // Additional stages for testing, SonarQube, Docker build, and deployment as in the previous script
    }
}

pipeline {
    agent any
    environment {
        AWS_ACCOUNT_ID="495381496358"
        AWS_DEFAULT_REGION="us-west-2"
	    CLUSTER_NAME="nodejs_test"
	    DESIRED_COUNT="1"
        IMAGE_REPO_NAME="nj"
        TASK_DEFINITION_NAME="nj_td_family"
        //Do not edit the variable IMAGE_TAG. It uses the Jenkins job build ID as a tag for the new image.
        IMAGE_TAG="${env.BUILD_ID}"
        //Do not edit REPOSITORY_URI.
        REPOSITORY_URI = "public.ecr.aws/s4g2j7e4/${IMAGE_REPO_NAME}"
        SUBNETS = "[subnet-0927b816ef9bc9334,subnet-041082b82472ee57b]"
        SECURITYGROUPS = "[sg-0c98a8cf5f771af81]"
    
    }
   
    stages {
        stage('checkout') {
            steps {
                git url: 'https://github.com/ramyagaraga/project.git',
                    branch: 'dev'
            }
        }

        // Building Docker image
        stage('Building image') {
            steps{
                script {
                    dockerImage = docker.build ("${IMAGE_REPO_NAME}:${IMAGE_TAG}")
                }
            }
        }
        // Uploading Docker image into AWS ECR
        stage('Releasing') {
            steps{  
                
                    sh 'aws ecr-public get-login-password --region us-east-1 | docker login --username AWS --password-stdin public.ecr.aws/s4g2j7e4'
                    sh "docker tag ${IMAGE_REPO_NAME}:${IMAGE_TAG} public.ecr.aws/s4g2j7e4/${IMAGE_REPO_NAME}:${IMAGE_TAG}" 
                    sh "docker push public.ecr.aws/s4g2j7e4/${IMAGE_REPO_NAME}:${IMAGE_TAG}"
                
            }
        }
        
        // Cluster creation
        stage('Create ECS Cluster') {
            steps {
                sh "aws ecs create-cluster --cluster-name ${CLUSTER_NAME} --region ${AWS_DEFAULT_REGION}"
            }
        }
        // Creating the Task Definition
        stage('Register Task Definition') {
            steps {
                script {
                    writeFile file: 'task-definition.json', text: """
                    {
                        "containerDefinitions": [
                            {
                                "name": "nodejs_container",
                                "image": "${REPOSITORY_URI}:${IMAGE_TAG}",
                                "essential": true, 
                                "cpu": 256,
                                "memory": 512,
                                "portMappings": [
                                    {
                                        "containerPort": 3000,
                                        "hostPort": 3000
                                    }
                                ]
                            }
                        ],
						"family": "${TASK_DEFINITION_NAME}", 
						"taskRoleArn": "arn:aws:iam::495381496358:role/ecsTaskExecutionRole",
					    "executionRoleArn": "arn:aws:iam::495381496358:role/ecsTaskExecutionRole",
						"requiresCompatibilities": ["FARGATE"],
						"networkMode": "awsvpc",
						"cpu": "256",
						"memory": "512"
                    }
                    """
                    sh "aws ecs register-task-definition --cli-input-json file://task-definition.json --region ${AWS_DEFAULT_REGION} "
                    
                }
            }
        }
        // Run the task
        stage('Run Task in ECS') {
            steps {
                sh'''
                aws ecs run-task --cluster ${CLUSTER_NAME} \
                 --task-definition ${TASK_DEFINITION_NAME} \
                 --launch-type="FARGATE" \
                 --network-configuration awsvpcConfiguration={subnets=[subnet-0927b816ef9bc9334,subnet-041082b82472ee57b],securityGroups=[sg-0c98a8cf5f771af81],assignPublicIp=ENABLED} \
                 --count 1
                 '''
            }
        }
    }
        // Clear local image registry. Note that all the data that was used to build the image is being cleared.
        // For different use cases, one may not want to clear all this data so it doesn't have to be pulled again for each build.
    post {
        always {
        sh 'docker system prune -a -f'
       }
    }
  } 
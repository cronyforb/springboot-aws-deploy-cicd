# springboot-aws-deploy

This is a sample microservice to deploy it on AWS ECS.

To build automated AWS CodePipeline and deploy microservice to AWS ECS, follow tutorial as shown in video :

Video Link :https://youtu.be/ARGmrYFfv44

Health Check command for AWS Task definition : 
```
CMD-SHELL,curl -f http://localhost:8080/actuator/health || exit 1
```


Prerequisite :
1. AWS acconunt.
2. Git and docker installed on the machine.
3. Docker should be started before building docker image.
4. And your favourite code editor 

1. ECR Repository Setup
bash
# Create ECR repository
aws ecr create-repository --repository-name springboot-app --region us-east-1

# Get ECR login command (use this in buildspec)
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin 147997122361.dkr.ecr.us-east-1.amazonaws.com
2. IAM Roles and Permissions
Add these permissions to your codebuild-code-build-service-role:

AmazonEC2ContainerRegistryFullAccess

AmazonEC2ContainerRegistryPowerUser

3. Build Specification (buildspec.yml)
yaml
version: 0.2

phases:
  pre_build:
    commands:
      - echo Logging in to Amazon ECR...
      - aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin 147997122361.dkr.ecr.us-east-1.amazonaws.com
      - COMMIT_HASH=$(echo $CODEBUILD_RESOLVED_SOURCE_VERSION | cut -c 1-7)
      - IMAGE_TAG=${COMMIT_HASH:=latest}

  build:
    commands:
      - echo Building the Docker image...
      - docker build -t springboot-app:$IMAGE_TAG .
      - docker tag springboot-app:$IMAGE_TAG 147997122361.dkr.ecr.us-east-1.amazonaws.com/springboot-app:$IMAGE_TAG

  post_build:
    commands:
      - echo Pushing the Docker image to ECR...
      - docker push 147997122361.dkr.ecr.us-east-1.amazonaws.com/springboot-app:$IMAGE_TAG
      - echo Writing image definitions file...
      - printf '[{"name":"springboot-container","imageUri":"%s"}]' 147997122361.dkr.ecr.us-east-1.amazonaws.com/springboot-app:$IMAGE_TAG > imagedefinitions.json

artifacts:
  files:
    - imagedefinitions.json
    - appspec.yml
    - taskdef.json
4. ECS Setup
Create Cluster:
Navigate to ECS console

Create cluster (Fargate recommended)

Note the cluster name

Create Task Definition:
Task definition name: springboot-task

Container name: springboot-container (update this in your pipeline)

Image: 147997122361.dkr.ecr.us-east-1.amazonaws.com/springboot-app:latest

Port mappings: Container port 8080

Health check: Adjust as needed for your Spring Boot app

Create Service:
Service name: springboot-service

Connect to load balancer (if needed)

Ensure proper security group is created and allows port 8080

5. CodePipeline Setup
Pipeline Structure:
Source Stage: CodeCommit repository

Build Stage: CodeBuild project

Deploy Stage: CodeDeploy to ECS

CodeBuild Project:
Use the buildspec.yml above

Environment: Managed image, Ubuntu, Docker support

Service role: codebuild-code-build-service-role

6. Testing Your Application
After deployment, test your application at:

text
http://<public-ip>:8080/demo/data
Important Notes:
Update the container name in all configurations to match your actual container name

Ensure your Spring Boot application has proper health check endpoints

Verify security groups allow traffic on port 8080

The ECR repository URL should match your actual account ID and region

This setup will automatically build your Docker image, push it to ECR, and deploy to ECS whenever you push code to your CodeCommit repositor

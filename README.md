# QRCODE project to deploy an sample html application to ECS using blue-green deployment with code build and code deploy.

This project consists of a sample QRcode application with cloud formation stack for running the Application in ECS with FARGATE capability and setting up code pipeline to trigger blue-green deployment.

In master branch the code consists of all the files required for Deploying in blue-green deployment.
In feature/rolling has the code for ECS standard deployments. (Work in progress)
The project is code pipeline, and code build uses Github/bitbucket as the source code.
Which means we need to set up connections for Github/bitbucket.
Go to code pipeline page under settings connections create a connection for you GITHUB/BITBUCKET Repository.



# ECS FARGATE BLUE GREEN DEPLOYMENT
Follow the he below steps to create a complete blue-green deployment.
1. Create ECR stack for the Application.
2. Create ECS Cluster stack.
3. Create Task definition for the Application.
4. Create taskdef.json in the root of source code Repository.
5. Create buildspec.yml and appspec.yaml in the root of source code repository.
6. Create ECS service for the Application
7. Create S3 bucket for code pipeline.
8. Create CodeBuild Stack.
9. Manually Create Deployment group in CodeDeploy.
10. Create CodePipeline stack.

### 1. Create ECR stack for the Application.
Run the cloudformation/ecr-registry.yml stack to create the ECR for the QRcode Application.
The registry id is exported as an output that is referenced cloudformation/code-build.yml

### 2. Create ECS Cluster stack.
Run the cloudformation/ecs-cluster.yml stack to create the ECS cluster.
The ECS Cluster id is exported as an output.

### 3. Create Task definition for the Application.
Run the cloudformation/qrcodetaskdefinition.yml stack to create the task definition for the Application.
The Tasks definition family and revision are exported as outputs which are referenced in cloudformation/ecs-service-bluegreen.yml.
This would have created the task definition in ECS.

### 4. Create taskdef.json in the root of source code Repository.
Once the task definition is created, it would be registered, and the revision will be active.
Go to the source code repository and create the taskdef.json with all task definition details update  the image name in the container definition to "image": "<IMAGE1_NAME>."

This will ensure  code pipeline using the latest image name created from the code build outputs for the deployment.

### 5. Create buildspec.yml and appspec.yaml in the root of source code repository.
Create the buildspec.yml in root folder of source code repository. The code build uses the build spec file to create the docker image and push to ECR, and when triggered from the pipeline, the output imageDetail.json,appspec.yaml,taskdef.json are stored in code pipeline S3 bucket under source artefacts.

In the appspec.YAML update the task definition property to TaskDefinition: <TASK_DEFINITION>.
Code pipeline ensures the new revision of task definition is created during the deployment with the updated. image details

The appspec.yaml,taskdef.json  are the same files in source code copied, however imageDetail.json will image uri of the latest docker image created.

##### NOTE:  
- The container name in appspec.yml should be same as the name of the container in the taskdef.json as well as the task definition created

### 6. Create ECS service for the Application
Run the cloudformation/ecs-service-bluegreen.yml stack to ECS service of type FARGATE and deployment controller as CODE_DEPLOY(Blue-Green Deployment).

ECS blue-green deployment requires a load balancer and two target groups for shifting the traffic during the deployment.
The stack will create a load balancer and two target groups and attach one of the target group to the load balancer. The primary target group is attached to the listener. The health check required for the container is specified in the target groups.

Up on create the ELB and Target groups, the service is created and linked to the load balancer.

In the cloud formation, I have used existing subnets and security groups if new security groups have to use the resource should be created.

### 7. Create S3 bucket for code pipeline.
The Codepipline uses an S3 bucket to store the source artefacts and build artefacts.
To create an S3 bucket in using cloudformation/codepipelines3.yml

##### NOTE:   

-    The bucket is used in code build stack, so that code build role has a policy attached to push the build artefacts to the S3 bucket.
-   The bucket is used in code pipeline stack, so that code build role has a policy attached to push the source artefacts to the S3 bucket.

### 8. Create CodeBuild Stack.
Run the cloudformation/codebuild.yml stack to create the code build for the Application.
The code build uses the build spec file to create the docker image and push to ECR, and when triggered from the pipeline, the outputs imageDetail.json,appspec.yaml,taskdef.json are stored in code pipeline S3 bucket under source artefacts.

The code pipeline references code build arn and name from the outputs if code build stack.

### 9. Manually Create Deployment group in CodeDeploy.

The blue-green deployment happens through the deployment group in the code deploy. I haven't found cloud formation for the creating deployment group for ECS in code deploy. So I have created this manually.

The above is an improvement which I am looking forward.

Create an application in the code deploy of compute type Amazon ECS.
Create a deployment group and configure the below details:
- Deployment group name
- ServiceRole
- ECS cluster name under Environment configuration
- Load balancer of the service
- Target group 1 name of the service
- Target group 2 name of the service
- Deployment settings
- 1. The Original revision termination configuration will allow to specify the time when to terminate the existing tasks.

Update the Application name and Deployment Group parameters for the code pipeline stack.

### 9.Create CodePipeline stack..
Run the cloudformation/codepipeline.yml stack to create the CodePipeline for the Application.
The code pipeline stack uses the policy for GITHUB connection.
Update the appropriate CODESTAR CONNECTION ARN   policy in the cloudformation/codepipeline.yml.
The pipeline triggers the code build code and deploy upon any changes to the source code.



# note:
- Please make sure the variables and stack names are updated appropriately in the stacks.

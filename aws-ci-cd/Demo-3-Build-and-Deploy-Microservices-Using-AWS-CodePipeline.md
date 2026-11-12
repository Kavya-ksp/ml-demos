# Demo 3 – Build and Deploy Microservices Using AWS CodePipeline

**Project:** Student Microservices Deployment Project  
**Level:** Beginner to Intermediate  
**Suggested duration:** 60–90 minutes  
**Mode:** Individual hands-on lab  
**AWS services:** AWS CodePipeline, AWS CodeBuild, Amazon ECR, Amazon ECS with AWS Fargate, Amazon CloudWatch Logs, AWS IAM  
**Source control:** GitHub

---

## 1. Objective

In this project, you will take a small, publicly available Node.js microservices application, containerize its services, and deploy them to Amazon ECS using an automated AWS CodePipeline.

You will deploy two services:

| Microservice | Port | Example responsibility |
|---|---:|---|
| `customer-service` | 3000 | Customer-facing supplier information |
| `employee-service` | 3001 | Employee/admin supplier management |

The application source and Dockerfiles are available in this public GitHub repository:

**Repository:** <https://github.com/cam-spec/aws-microservices-cicd-platform>

The repository contains two Node.js service folders under `services/`, each with its own Dockerfile. The service endpoints used in this lab are listed in the verification section.

> **Important learning note:** To keep this first container-based pipeline manageable, both containers will run in one ECS Fargate task and be updated by one ECS service. They are separate application services and separate containers, but they are deployed and scaled together in this lab. In a production microservices design, each independently scalable service would normally have its own ECS service/task definition, with traffic routed using an Application Load Balancer or service discovery.

## 2. Architecture

```text
Developer commits code to GitHub
                |
                v
        AWS CodePipeline
                |
                v
          Source stage
                |
                v
          Build stage
          AWS CodeBuild
       (Docker build x 2)
                |
         +------+------+
         |             |
         v             v
  Amazon ECR      Amazon ECR
 customer-service employee-service
         |             |
         +------+------+
                |
                v
         BuildArtifact
     imagedefinitions.json
                |
                v
          Deploy stage
      AWS CodePipeline ECS action
                |
                v
        Amazon ECS / Fargate
          One ECS service
          One task definition
          +--------------------+
          | customer-service   | :3000
          | employee-service   | :3001
          +--------------------+
                |
                v
       Public task IP for lab testing
```

**What each AWS service does**

| Service | Responsibility |
|---|---|
| GitHub | Stores the application source code and build instructions. |
| CodePipeline | Orchestrates Source → Build → Deploy. |
| CodeBuild | Builds both Docker images and pushes them to ECR. |
| Amazon ECR | Stores the two Docker images. |
| Amazon ECS with Fargate | Runs the two containers without managing EC2 servers. |
| CloudWatch Logs | Stores application container logs. |
| IAM | Controls what CodeBuild, CodePipeline and ECS are allowed to do. |

## 3. Prerequisites

Before starting, make sure you have:

- An AWS account or an AWS lab account with permission to create the listed resources and IAM roles/policies.
- A GitHub account.
- Access to the AWS CodeConnections GitHub App flow.
- An AWS Region in which CodeBuild, ECR and ECS Fargate are available.
- A default VPC with public subnets, or an existing VPC with public subnets and an Internet Gateway.

**Use the same AWS Region** for the ECR repositories, CodeBuild project, CodePipeline, ECS cluster and CloudWatch log group.

This lab uses the default VPC to reduce networking setup. If your training account does not have a default VPC, ask the instructor to provide a VPC with public subnets and an Internet Gateway before you begin.

### Cost and security warning

ECS Fargate, CodeBuild, ECR image storage and CloudWatch Logs can incur charges. Complete the lab promptly and perform the cleanup steps at the end. The security group in this lab opens application ports only to your current public IP address. Do not use these settings as a production network design.

---

# Part A – Prepare the GitHub project

## Step 1. Fork the public repository

1. Open <https://github.com/cam-spec/aws-microservices-cicd-platform>.
2. Select **Fork**.
3. Choose your GitHub account as the owner.
4. Keep the repository name `aws-microservices-cicd-platform` or choose a new name.
5. Create the fork.
6. Open your fork and verify that the default branch is `main`.

Use your fork for this lab so that you can make a small code change later and trigger your own pipeline.

> Review the upstream repository's current licence and usage terms before reusing or redistributing its source outside this educational exercise.

## Step 2. Review the application structure

In your fork, verify that these folders and files exist:

```text
aws-microservices-cicd-platform/
|
+-- services/
|   +-- customer-service/
|   |   +-- Dockerfile
|   |   +-- index.js
|   |   +-- package.json
|   |
|   +-- employee-service/
|       +-- Dockerfile
|       +-- index.js
|       +-- package.json
|
+-- docker-compose.yml
+-- README.md
```

The two Dockerfiles are already provided. You do not need to create Dockerfiles for this exercise.

## Step 3. Add a root-level buildspec.yml

The upstream repository includes a buildspec under `cicd/`. For this lab, create a **new `buildspec.yml` in the repository root** so that the CodeBuild configuration is easy to locate and the ECS deployment file is produced at the root of the build artifact.

1. In your fork, select **Add file → Create new file**.
2. Enter the filename `buildspec.yml`.
3. Paste the following content exactly:

```yaml
version: 0.2

env:
  variables:
    ECR_CUSTOMER_REPOSITORY: "customer-service"
    ECR_EMPLOYEE_REPOSITORY: "employee-service"

phases:
  pre_build:
    commands:
      - echo "Logging in to Amazon ECR..."
      - export AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
      - export ECR_REGISTRY=${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com
      - export IMAGE_TAG=$(echo ${CODEBUILD_RESOLVED_SOURCE_VERSION} | cut -c 1-7)
      - aws ecr get-login-password --region ${AWS_DEFAULT_REGION} | docker login --username AWS --password-stdin ${ECR_REGISTRY}

  build:
    commands:
      - echo "Building customer-service image..."
      - docker build -t ${ECR_REGISTRY}/${ECR_CUSTOMER_REPOSITORY}:${IMAGE_TAG} -t ${ECR_REGISTRY}/${ECR_CUSTOMER_REPOSITORY}:latest ./services/customer-service
      - echo "Building employee-service image..."
      - docker build -t ${ECR_REGISTRY}/${ECR_EMPLOYEE_REPOSITORY}:${IMAGE_TAG} -t ${ECR_REGISTRY}/${ECR_EMPLOYEE_REPOSITORY}:latest ./services/employee-service

  post_build:
    commands:
      - echo "Pushing customer-service images..."
      - docker push ${ECR_REGISTRY}/${ECR_CUSTOMER_REPOSITORY}:${IMAGE_TAG}
      - docker push ${ECR_REGISTRY}/${ECR_CUSTOMER_REPOSITORY}:latest
      - echo "Pushing employee-service images..."
      - docker push ${ECR_REGISTRY}/${ECR_EMPLOYEE_REPOSITORY}:${IMAGE_TAG}
      - docker push ${ECR_REGISTRY}/${ECR_EMPLOYEE_REPOSITORY}:latest
      - echo "Creating ECS image definitions file..."
      - printf '[{"name":"customer-service","imageUri":"%s"},{"name":"employee-service","imageUri":"%s"}]' ${ECR_REGISTRY}/${ECR_CUSTOMER_REPOSITORY}:${IMAGE_TAG} ${ECR_REGISTRY}/${ECR_EMPLOYEE_REPOSITORY}:${IMAGE_TAG} > imagedefinitions.json
      - cat imagedefinitions.json

artifacts:
  files:
    - imagedefinitions.json
```

4. Commit the file to the `main` branch.

**What this buildspec does**

1. Logs in to Amazon ECR.
2. Creates a unique image tag from the Git commit ID.
3. Builds the customer-service Docker image.
4. Builds the employee-service Docker image.
5. Pushes both images to their ECR repositories using the commit tag and `latest` tag.
6. Generates `imagedefinitions.json` with both container names and image URIs.
7. Publishes `imagedefinitions.json` as the CodeBuild output artifact.

The two `name` values in `imagedefinitions.json` must match the container names in the ECS task definition exactly. The ECS standard deployment action supports multiple container/image entries in this file.

---

# Part B – Create Amazon ECR repositories

## Step 4. Create the customer-service repository

1. Sign in to the AWS Management Console.
2. Search for **Amazon ECR** and open it.
3. In the navigation pane, choose **Private registry → Repositories**.
4. Choose **Create repository**.
5. Configure:

| Setting | Value |
|---|---|
| Visibility | Private |
| Repository name | `customer-service` |
| Image tag mutability | Mutable |
| Image scan on push | Optional; enable if available |
| Encryption | Default AES-256 |

6. Choose **Create repository**.

## Step 5. Create the employee-service repository

Repeat Step 4 with this repository name:

`employee-service`

The repository names must match the values in `buildspec.yml` exactly.

At this stage, both repositories are empty. CodeBuild will push the images in a later step.

---

# Part C – Create the Source and Build pipeline

We will first create a pipeline with Source and Build stages. This first execution will build and push both images to ECR. After that, we will create the ECS resources and add the Deploy stage.

## Step 6. Create a CodePipeline

1. Open **AWS CodePipeline**.
2. Choose **Create pipeline**.
3. If asked for a creation option, choose **Build custom pipeline**.
4. Configure the pipeline:

| Setting | Value |
|---|---|
| Pipeline name | `Student-Microservices-Pipeline` |
| Pipeline type | V2 (or the default offered) |
| Execution mode | Default |
| Service role | New service role |
| Artifact store | Default location |
| Encryption key | Default AWS managed key |

5. Continue to the Source stage.

## Step 7. Configure GitHub as the Source

1. Select **GitHub (via GitHub App)**.
2. Choose **Connect to GitHub** or select an existing connection.
3. If creating a connection:
   - Enter a connection name such as `Student-Microservices-GitHub`.
   - Authorize the AWS Connector for GitHub.
   - Install the GitHub App for your account.
   - Grant it access to your fork of the repository.
   - Return to AWS and complete the connection.
4. Select your GitHub connection.
5. Select your forked repository.
6. Select branch `main`.
7. Keep the default ZIP/source artifact format. Do not choose Full clone for this lab.
8. Name the source output artifact `SourceArtifact`, if the wizard asks for the artifact name.
9. Continue to the Build stage.

## Step 8. Create the CodeBuild project from the pipeline wizard

1. Select **AWS CodeBuild** as the Build provider.
2. Choose **Create project**.
3. Enter the project name:

`Student-Microservices-Build`

4. Configure the build environment:

| Setting | Value |
|---|---|
| Environment image | Managed image |
| Operating system | Amazon Linux |
| Runtime | Standard |
| Image | Latest supported Standard image |
| Architecture | x86_64 |
| Compute type | EC2 Small |
| Service role | New service role |

5. Expand **Additional configuration** or **Environment** settings.
6. Enable **Privileged** mode. This is required because CodeBuild must run Docker commands to build container images.
7. Under Buildspec, select **Use a buildspec file**.
8. Enter `buildspec.yml` as the buildspec filename. This is the root-level file you added in Step 3.
9. Enable CloudWatch Logs and keep the default log group/stream settings.
10. Choose **Continue to CodePipeline**.

Back in the CodePipeline wizard, verify the Build action:

| Setting | Value |
|---|---|
| Project | `Student-Microservices-Build` |
| Input artifact | `SourceArtifact` |
| Output artifact | `BuildArtifact` |

The source ZIP is passed to CodeBuild. CodeBuild builds and pushes the images to ECR. The `BuildArtifact` contains `imagedefinitions.json`, which will be used by the ECS Deploy action later.

## Step 9. Skip the Deploy stage for now

If the wizard asks you to configure a Deploy stage, choose **Skip deploy stage**. We will add the ECS deployment after the first build has populated ECR and after the ECS service exists.

Review and create the pipeline.

## Step 10. Give CodeBuild permission to push to ECR

The CodeBuild service role needs permissions to authenticate to ECR and push the two images.

1. Open **AWS CodeBuild → Build projects**.
2. Open `Student-Microservices-Build`.
3. Locate the service role in the project details or environment configuration and open the role in IAM.
4. In IAM, choose **Add permissions → Attach policies**.
5. Search for and attach the AWS managed policy:

`AmazonEC2ContainerRegistryPowerUser`

6. Save the change.

This managed policy is convenient for a classroom exercise. In a production account, replace it with a least-privilege policy scoped to the two ECR repositories.

The CodeBuild service role created by the wizard should also retain its normal CodeBuild permissions for CloudWatch Logs and the CodePipeline artifact store. Do not replace or remove those permissions.

## Step 11. Run the Source and Build stages

1. Return to CodePipeline.
2. Open `Student-Microservices-Pipeline`.
3. If the initial execution started before you attached the ECR policy and failed, choose **Release change** to start a new execution.
4. Wait for Source and Build to succeed.
5. Open the Build action details and review the CodeBuild logs.
6. Confirm that the build logs show both Docker images being built and pushed.

The build should create these ECR repositories and tags:

```text
customer-service:
  latest
  <7-character-commit-id>

employee-service:
  latest
  <7-character-commit-id>
```

7. Open Amazon ECR and verify that both repositories contain images.

**Do not continue until both ECR repositories contain at least one image.** The ECS task definition created next will refer to the `latest` tag.

---

# Part D – Prepare Amazon ECS and Fargate

## Step 12. Create or verify the ECS task execution role

ECS Fargate needs an execution role to pull private images from ECR and send container logs to CloudWatch Logs.

1. Open **IAM → Roles**.
2. Search for `ecsTaskExecutionRole`.
3. If the role already exists and has the `AmazonECSTaskExecutionRolePolicy` attached, use it.
4. Otherwise, choose **Create role**.
5. Select:
   - Trusted entity type: **AWS service**
   - Service/use case: **Elastic Container Service → Elastic Container Service Task**
6. Attach the managed policy `AmazonECSTaskExecutionRolePolicy`.
7. Name the role `ecsTaskExecutionRole` and create it.

Do not use the CodeBuild role or the CodePipeline role as the ECS task execution role.

## Step 13. Create an ECS cluster

1. Open **Amazon ECS**.
2. Choose **Clusters → Create cluster**.
3. Enter:

`Student-Microservices-Cluster`

4. Select the AWS Fargate/serverless infrastructure option (do not create EC2 container instances).
5. Keep the default settings and create the cluster.

## Step 14. Create a CloudWatch log group

1. Open **CloudWatch → Logs → Log groups**.
2. Choose **Create log group**.
3. Enter:

`/ecs/student-microservices`

4. Create the log group.

This log group will receive logs from both containers.

## Step 15. Create the ECS task definition

The task definition describes the containers, images, ports, CPU/memory and execution role.

1. In Amazon ECS, open **Task definitions**.
2. Choose **Create new task definition**.
3. Configure the task:

| Setting | Value |
|---|---|
| Task definition family | `student-microservices-task` |
| Launch type / infrastructure | AWS Fargate |
| Operating system | Linux |
| CPU architecture | x86_64 |
| Network mode | awsvpc |
| Task CPU | 0.5 vCPU (512 units) |
| Task memory | 1 GB (1024 MiB) |
| Task execution role | `ecsTaskExecutionRole` |
| Task role | None for this sample |

4. Add the first container.

**Container 1 – customer-service**

| Setting | Value |
|---|---|
| Container name | `customer-service` |
| Image URI | `<ACCOUNT_ID>.dkr.ecr.<REGION>.amazonaws.com/customer-service:latest` |
| Essential | Yes |
| Port mapping | Container port `3000`, protocol TCP |
| Memory reservation | 256 MiB (if the console asks) |

Replace `<ACCOUNT_ID>` with your AWS account ID and `<REGION>` with the Region where you created ECR repositories. You can copy the complete image URI from the ECR repository page.

5. Configure logging for this container:
   - Log driver: `awslogs`
   - Log group: `/ecs/student-microservices`
   - Region: your selected AWS Region
   - Stream prefix: `customer`

6. Add the second container.

**Container 2 – employee-service**

| Setting | Value |
|---|---|
| Container name | `employee-service` |
| Image URI | `<ACCOUNT_ID>.dkr.ecr.<REGION>.amazonaws.com/employee-service:latest` |
| Essential | Yes |
| Port mapping | Container port `3001`, protocol TCP |
| Memory reservation | 256 MiB (if the console asks) |

7. Configure logging for this container:
   - Log driver: `awslogs`
   - Log group: `/ecs/student-microservices`
   - Region: your selected AWS Region
   - Stream prefix: `employee`

8. Review the task definition. Confirm that the container names are exactly `customer-service` and `employee-service`.
9. Create the task definition.

> **Important:** The names are case-sensitive and must match the `name` properties generated in `imagedefinitions.json`. The image URIs must point to the two ECR repositories created in Step 4 and Step 5.

## Step 16. Create a security group for the ECS task

1. Open **VPC → Security Groups**.
2. Choose **Create security group**.
3. Enter:

| Setting | Value |
|---|---|
| Security group name | `student-microservices-sg` |
| Description | Allow student test access to two microservices |
| VPC | Default VPC (or your instructor-provided VPC) |

4. Add these inbound rules:

| Type | Port | Source |
|---|---:|---|
| Custom TCP | 3000 | My IP |
| Custom TCP | 3001 | My IP |

5. Keep the default outbound rule allowing all traffic.
6. Create the security group.

Use **My IP** for both inbound rules. If your classroom network uses a shared public IP, your instructor may provide that IP range. Avoid opening these ports to `0.0.0.0/0` unless the instructor explicitly authorizes a brief, controlled test.

## Step 17. Create the ECS service

1. Open **Amazon ECS → Clusters**.
2. Select `Student-Microservices-Cluster`.
3. Open the **Services** tab.
4. Choose **Create**.
5. Configure the service:

| Setting | Value |
|---|---|
| Compute options | Launch type |
| Launch type | Fargate |
| Task definition family | `student-microservices-task` |
| Revision | Latest revision |
| Service name | `Student-Microservices-Service` |
| Desired tasks | `1` |
| Deployment type | Rolling update / default |

6. Under Networking:
   - Select the default VPC (or the VPC provided by your instructor).
   - Select public subnets.
   - Select `student-microservices-sg`.
   - Turn **Auto-assign public IP** on.
7. Do not configure a load balancer for this introductory exercise.
8. Create the service.
9. Wait until the service shows one running task and the deployment is stable.

If the task does not start, open the task details and inspect the stopped reason. Also check CloudWatch Logs. Common causes include an incorrect ECR image URI, a missing execution role, or a Region mismatch.

## Step 18. Verify the initial ECS deployment

1. In the ECS service, open the **Tasks** tab.
2. Select the running task.
3. Open its **Networking** details.
4. Copy the task's public IPv4 address.
5. Test these URLs in a browser, replacing `<PUBLIC_IP>` with the task IP:

**Customer service**

- `http://<PUBLIC_IP>:3000/`
- `http://<PUBLIC_IP>:3000/health`
- `http://<PUBLIC_IP>:3000/suppliers`

**Employee service**

- `http://<PUBLIC_IP>:3001/`
- `http://<PUBLIC_IP>:3001/health`
- `http://<PUBLIC_IP>:3001/admin/suppliers`

Expected responses include:

- `Customer Microservice Running`
- `Employee Microservice Running`
- Health endpoints returning JSON with `status: ok`.

The application source uses these routes. Do not use `/customers` or `/employees` for this particular sample; those paths are not the implemented routes in the service files.

---

# Part E – Add the ECS Deploy stage to CodePipeline

## Step 19. Give CodePipeline permission to deploy to ECS

The CodePipeline service role needs permissions to register a new task-definition revision and update the ECS service.

1. Open **CodePipeline** and open `Student-Microservices-Pipeline`.
2. Locate the pipeline's **Service role ARN** in the pipeline details/settings.
3. Open **IAM → Roles** and find that exact role.
4. Choose **Add permissions → Create inline policy**.
5. Select the JSON editor and paste the policy below.
6. Replace `<REGION>`, `<ACCOUNT_ID>`, `<CLUSTER_NAME>`, `<SERVICE_NAME>` and `<TASK_DEFINITION_FAMILY>` with your values.
7. Choose a policy name such as `AllowStudentECSDeployment` and save it.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "RegisterAndDescribeTaskDefinitions",
      "Effect": "Allow",
      "Action": [
        "ecs:DescribeTaskDefinition",
        "ecs:RegisterTaskDefinition"
      ],
      "Resource": "*"
    },
    {
      "Sid": "UpdateStudentECSService",
      "Effect": "Allow",
      "Action": [
        "ecs:DescribeServices",
        "ecs:UpdateService"
      ],
      "Resource": "arn:aws:ecs:<REGION>:<ACCOUNT_ID>:service/<CLUSTER_NAME>/<SERVICE_NAME>"
    },
    {
      "Sid": "TagNewTaskDefinition",
      "Effect": "Allow",
      "Action": "ecs:TagResource",
      "Resource": "arn:aws:ecs:<REGION>:<ACCOUNT_ID>:task-definition/<TASK_DEFINITION_FAMILY>:*",
      "Condition": {
        "StringEquals": {
          "ecs:CreateAction": "RegisterTaskDefinition"
        }
      }
    },
    {
      "Sid": "PassTaskExecutionRoleToECS",
      "Effect": "Allow",
      "Action": "iam:PassRole",
      "Resource": "arn:aws:iam::<ACCOUNT_ID>:role/ecsTaskExecutionRole",
      "Condition": {
        "StringEquals": {
          "iam:PassedToService": [
            "ecs.amazonaws.com",
            "ecs-tasks.amazonaws.com"
          ]
        }
      }
    }
  ]
}
```

The pipeline's original service role permissions for its GitHub connection, CodeBuild action and artifact bucket must remain attached. This inline policy adds ECS deployment permissions; it does not replace the existing permissions.

If your instructor account uses a different execution-role name, update the `iam:PassRole` resource ARN to match the role selected in the task definition.

## Step 20. Add a Deploy stage

1. Return to **AWS CodePipeline**.
2. Open `Student-Microservices-Pipeline`.
3. Choose **Edit**.
4. Add a new stage after Build.
5. Name the stage `Deploy`.
6. Add an action and configure:

| Setting | Value |
|---|---|
| Action name | `DeployToECS` |
| Action provider | Amazon ECS |
| Region | Your selected AWS Region |
| Input artifact | `BuildArtifact` |
| Cluster name | `Student-Microservices-Cluster` |
| Service name | `Student-Microservices-Service` |
| Image definitions file | `imagedefinitions.json` |

7. Do not add an output artifact for this ECS deploy action.
8. Save the pipeline changes.

**Critical artifact mapping**

| Stage | Artifact |
|---|---|
| Source output | `SourceArtifact` |
| Build input | `SourceArtifact` |
| Build output | `BuildArtifact` |
| Deploy input | `BuildArtifact` |

The ECS Deploy action reads `imagedefinitions.json` from `BuildArtifact`. The file contains both container names and their new ECR image URIs.

## Step 21. Run the complete pipeline

1. Open the pipeline.
2. Choose **Release change** (or the available start-execution option) to run the complete pipeline.
3. Observe the stages:

| Stage | Expected result |
|---|---|
| Source | Succeeded |
| Build | Succeeded |
| Deploy | Succeeded |

4. Open the Build action logs. Confirm that both images were pushed and that `imagedefinitions.json` was generated.
5. Open ECS and inspect the service deployment. CodePipeline should have registered a new task-definition revision with the image tags from the build.
6. Wait until the ECS service is stable and one task is running.

If the Deploy stage fails with an IAM error, re-check the CodePipeline service role policy from Step 19. If it fails because the image definitions file is missing, check the CodeBuild build artifact and confirm that `imagedefinitions.json` is at the root of the `BuildArtifact`.

---

# Part F – Test automatic microservices deployment

## Step 22. Make a source-code change

1. Open your forked GitHub repository.
2. Navigate to `services/customer-service/index.js`.
3. Find this line:

```javascript
res.send("Customer Microservice Running");
```

4. Change it to:

```javascript
res.send("Customer Microservice Running on AWS ECS!");
```

5. Commit the change to the `main` branch.

## Step 23. Observe the automatic pipeline execution

1. Return to AWS CodePipeline.
2. GitHub should trigger a new pipeline execution for the commit.
3. Observe Source, Build and Deploy.
4. Confirm that CodeBuild builds and pushes both images with the new commit tag.
5. Confirm that the ECS Deploy action updates the task definition and ECS service.
6. Wait for the deployment to become stable.
7. Open the ECS service's **Tasks** tab and select the current running task.
8. Copy its current public IPv4 address. The public IP can change when ECS replaces a task during deployment.
9. Open:

`http://<NEW_PUBLIC_IP>:3000/`

Expected response:

`Customer Microservice Running on AWS ECS!`

Also verify the employee service:

`http://<NEW_PUBLIC_IP>:3001/`

Expected response:

`Employee Microservice Running`

**Demo completed:** A GitHub commit triggered CodePipeline, CodeBuild rebuilt both microservice images, ECR stored the images, and the ECS deployment action updated the running Fargate service.

---

# Part G – Troubleshooting

| Issue | What to check |
|---|---|
| CodeBuild cannot run `docker build` | Confirm that Privileged mode is enabled for the CodeBuild environment. |
| ECR login fails | Confirm the CodeBuild service role has ECR permissions and the Region is correct. |
| ECR push fails with `RepositoryNotFoundException` | Verify that `customer-service` and `employee-service` repositories exist in the same Region. Check spelling and case. |
| Build cannot find Dockerfile | Confirm the repository contains `services/customer-service/Dockerfile` and `services/employee-service/Dockerfile`. |
| `imagedefinitions.json` is missing | Confirm the file is generated at the repository/build root and included under `artifacts.files` in `buildspec.yml`. |
| ECS Deploy reports invalid image definitions | Confirm the JSON is valid and container names match the ECS task definition exactly. |
| ECS Deploy returns `AccessDenied` | Check the CodePipeline service role inline policy and `iam:PassRole` permission. |
| ECS task fails to start | Check the task stopped reason, image URIs, `ecsTaskExecutionRole` and CloudWatch Logs. |
| Browser cannot reach port 3000/3001 | Confirm the task has a public IP, it is in a public subnet, Auto-assign public IP is enabled and the security group permits your IP on both ports. |
| One service works but the other does not | Check the corresponding container's port mapping, container logs and security-group rule. |
| Pipeline does not start after a commit | Check the GitHub connection, selected repository/branch and source trigger configuration. |

## Useful verification commands (optional)

If AWS CLI is configured, you can inspect the ECS service and task:

```bash
aws ecs describe-services \
  --cluster Student-Microservices-Cluster \
  --services Student-Microservices-Service \
  --region <REGION>
```

List tasks:

```bash
aws ecs list-tasks \
  --cluster Student-Microservices-Cluster \
  --service-name Student-Microservices-Service \
  --region <REGION>
```

These commands are optional. The AWS Console walkthrough is sufficient for the lab.

---

# Part H – Cleanup (Required)

Do not leave the ECS service running after the lab. Fargate tasks can continue to incur charges while running.

Perform cleanup in this order:

1. Open the ECS service `Student-Microservices-Service`.
2. Update desired tasks to `0` and wait for running tasks to stop.
3. Delete `Student-Microservices-Service`.
4. Delete `Student-Microservices-Cluster` after the service is removed.
5. Open ECR and delete the images and repositories `customer-service` and `employee-service` (force delete if the console asks to remove images).
6. Delete `Student-Microservices-Pipeline`.
7. Delete the CodeBuild project `Student-Microservices-Build`.
8. Delete the CloudWatch log group `/ecs/student-microservices`.
9. Delete the security group `student-microservices-sg` if it is no longer used.
10. Review IAM roles created only for this lab. Delete them only after confirming that no other resource uses them. Do not delete AWS service-linked roles.
11. Review the CodePipeline artifact S3 bucket. Delete it only if it was created for this lab, is empty, and is not used by another pipeline.
12. Remove the GitHub App connection only if it was created solely for this lab and is not shared with another pipeline.

Keep your default VPC and its subnets; they may be used by other AWS exercises.

---

# Student completion checklist

- [ ] Forked the public microservices repository.
- [ ] Added the root-level `buildspec.yml`.
- [ ] Created both ECR repositories with the exact names.
- [ ] Created the GitHub connection and Source stage.
- [ ] Created the CodeBuild project with Privileged mode enabled.
- [ ] Granted the CodeBuild role permission to push to ECR.
- [ ] Ran the Source and Build pipeline and verified both ECR images.
- [ ] Created the ECS task execution role.
- [ ] Created the ECS cluster and CloudWatch log group.
- [ ] Created a Fargate task definition with both containers and correct port mappings.
- [ ] Created the ECS service and verified both endpoints.
- [ ] Added the ECS Deploy stage with `BuildArtifact` as its input.
- [ ] Added the ECS permissions to the CodePipeline service role.
- [ ] Ran the complete pipeline successfully.
- [ ] Changed source code and verified automatic deployment.
- [ ] Cleaned up the AWS resources.

---

# Official AWS references

- [AWS CodeBuild – Build Docker images](https://docs.aws.amazon.com/codebuild/latest/userguide/sample-docker.html)
- [AWS CodeBuild – Create a build project and enable Privileged mode](https://docs.aws.amazon.com/codebuild/latest/userguide/create-project.html)
- [Amazon ECR – Creating a private repository](https://docs.aws.amazon.com/AmazonECR/latest/userguide/repository-create.html)
- [Amazon ECS – Task execution IAM role](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task_execution_IAM_role.html)
- [Amazon ECS – Creating a Fargate task definition](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task_definitions.html)
- [Amazon ECS – Creating an ECS service](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/create-service-console-v2.html)
- [CodePipeline – Amazon ECS standard deployment action](https://docs.aws.amazon.com/codepipeline/latest/userguide/action-reference-ECS.html)
- [CodePipeline – `imagedefinitions.json` format](https://docs.aws.amazon.com/codepipeline/latest/userguide/file-reference.html)


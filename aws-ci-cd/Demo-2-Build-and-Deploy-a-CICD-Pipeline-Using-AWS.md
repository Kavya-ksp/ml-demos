# Demo 2 – Build and Deploy a CI/CD Pipeline Using AWS

**Audience:** Undergraduate students  
**Level:** Beginner  
**Estimated duration:** 25–30 minutes  
**Mode:** Individual hands-on exercise (each participant follows the steps)  
**AWS services:** AWS CodePipeline, AWS CodeBuild, Amazon S3  
**Source control:** GitHub

## 1. Objective

In this exercise, you will create a simple CI/CD pipeline. The pipeline will retrieve application files from GitHub, run build commands using AWS CodeBuild, and deploy the build output to an Amazon S3 bucket.

By the end of the exercise, you will be able to:

- Connect a GitHub repository to AWS CodePipeline.
- Use AWS CodeBuild to run a buildspec file.
- Pass a build artifact from CodeBuild to the deployment stage.
- Deploy the application file to Amazon S3.
- Trigger a new pipeline execution by committing a change to GitHub.

## 2. Understand the workflow

```text
Developer commits code to GitHub
              |
              v
     AWS CodePipeline
          Source
              |
       SourceArtifact
              |
              v
      AWS CodeBuild
           Build
     Reads buildspec.yml
     Creates build output
              |
        BuildArtifact
              |
              v
     AWS CodePipeline
           Deploy
              |
              v
        Amazon S3
      index.html deployed
```

**Important:** The pipeline uses two different S3 locations:

1. **CodePipeline artifact store:** An S3 bucket used internally by CodePipeline to pass source and build artifacts between stages. Allow AWS to use the default artifact store when creating the pipeline.
2. **Application deployment bucket:** The S3 bucket you create in this exercise. The final `index.html` file is deployed here.

Do not use the application deployment bucket as the CodePipeline artifact store.

## 3. Prerequisites

Before you begin, make sure you have:

- An AWS account or an approved AWS learner-lab account.
- A GitHub account.
- Permission to create CodeBuild and CodePipeline projects and IAM service roles.
- Permission to create and use an S3 bucket.
- Access to the same AWS Region for CodePipeline, CodeBuild, and the deployment bucket.

Use an IAM user or role with the required permissions. Do not use the AWS account root user for routine work. Some classroom or learner-lab accounts restrict IAM role creation; ask your instructor to provide the required permissions or pre-created roles if you cannot create them.

> **Cost and security note:** AWS resources may incur charges. Keep the S3 bucket private, do not upload secrets, and delete the lab resources after the exercise. Check current AWS pricing and any learner-lab limits before starting.

---

# Part A – Prepare the application in GitHub

## Step 1. Create a GitHub repository

1. Sign in to GitHub.
2. Select **New repository**.
3. Enter the following details:

   | Setting | Value |
   |---|---|
   | Repository name | `aws-cicd-demo` |
   | Visibility | Public (for this classroom exercise) |
   | Initialize repository | Add a README file |
   | Default branch | `main` |

4. Select **Create repository**.

A public repository makes the sample application easy to access during the exercise. Never place passwords, access keys, tokens, or other confidential information in the repository.

## Step 2. Add the application file

1. In the repository, select **Add file** > **Create new file**.
2. Enter the filename `index.html`.
3. Paste the following content:

```html
<!DOCTYPE html>
<html>
<head>
    <title>My First AWS CI/CD Application</title>
</head>
<body>
    <h1>Welcome to AWS CI/CD!</h1>
    <h2>My First Automated Deployment</h2>
    <p>This application is deployed using AWS CodePipeline and AWS CodeBuild.</p>
    <p>Learning AWS DevOps is exciting!</p>
</body>
</html>
```

4. Select **Commit changes**.
5. Commit directly to the `main` branch.

## Step 3. Add the CodeBuild buildspec file

1. Select **Add file** > **Create new file**.
2. Enter the filename `buildspec.yml`.
3. Paste the following YAML. Keep the indentation as shown.

```yaml
version: 0.2

phases:
  build:
    commands:
      - echo "Starting application build"
      - test -f index.html
      - mkdir -p output
      - cp index.html output/index.html
      - echo "Application packaged successfully"

artifacts:
  base-directory: output
  files:
    - index.html
```

4. Select **Commit changes**.
5. Commit directly to the `main` branch.

Your repository should now contain:

```text
aws-cicd-demo/
├── index.html
├── buildspec.yml
└── README.md
```

### What does `buildspec.yml` do?

- `phases` defines the commands CodeBuild executes.
- `test -f index.html` checks that the application file exists. The build fails if it is missing.
- `mkdir -p output` creates the output folder.
- `cp` copies the application file into the output folder.
- `artifacts` tells CodeBuild which file to package for the next pipeline stage.
- `base-directory: output` places `index.html` at the root of the build artifact.

The artifact configuration is essential. The Deploy stage needs a build output artifact from CodeBuild.

---

# Part B – Create the application deployment bucket

## Step 4. Create an Amazon S3 bucket

1. Sign in to the AWS Management Console.
2. Select the AWS Region you will use for this exercise.
3. Search for **Amazon S3** and open the service.
4. Select **Create bucket**.
5. Enter the following settings:

   | Setting | Value |
   |---|---|
   | Bucket name | `student-cicd-deploy-<unique-id>` |
   | AWS Region | Same Region as the pipeline |
   | Object Ownership | Keep the default setting (ACLs disabled / Bucket owner enforced, if displayed) |
   | Block Public Access | Keep all settings enabled |
   | Bucket Versioning | Leave disabled for this exercise |
   | Default encryption | Keep enabled |

6. Replace `<unique-id>` with a unique combination of letters and numbers. S3 bucket names must be globally unique. Do not type the angle brackets.
7. Select **Create bucket**.

Keep this bucket private. This exercise deploys a file to S3; it does not configure public website hosting.

---

# Part C – Create the CodeBuild project for the pipeline

Create a separate CodeBuild project for this exercise. **Do not reuse the CodeBuild project from Demo 1 if it was configured with `No artifacts`.**

## Step 5. Create the CodeBuild project

1. In the AWS console, search for **AWS CodeBuild**.
2. Open **Build projects**.
3. Select **Create build project**.
4. Configure the project as follows.

### Project configuration

| Setting | Value |
|---|---|
| Project name | `Student-CICD-Build` |
| Description | `Build project used by Student-CICD-Pipeline` (optional) |

### Source

| Setting | Value |
|---|---|
| Source provider | **AWS CodePipeline** |
| Source version | Leave at the default / blank, if shown |

Do not select GitHub as the CodeBuild source provider. In this design, CodePipeline retrieves the GitHub files and passes the source artifact to CodeBuild.

### Environment

Use a managed Linux build environment. The exact image labels can vary as AWS updates its supported images.

| Setting | Value |
|---|---|
| Provisioning model | On-demand |
| Environment image | Managed image |
| Operating system | Ubuntu or another available Linux Standard image |
| Runtime | Standard |
| Image version | Latest supported Standard image |
| Compute | EC2 – Small (or the smallest available suitable compute option) |
| Service role | New service role |

If the console asks for an environment type or architecture, choose a standard Linux x86-64 environment.

### Buildspec

1. Select **Use a buildspec file**.
2. Use the default buildspec filename: `buildspec.yml`.
3. The file must be in the root of the GitHub repository.

### Artifacts

Set the artifact type to **CodePipeline** if the console displays an artifact type selection for this project.

- Do **not** select **No artifacts**.
- Do **not** select Amazon S3 as a separate CodeBuild output bucket for this exercise.
- CodePipeline will manage the input and output artifact transfer for this build action.

### Logs

Enable CloudWatch Logs and keep the default log group and stream settings, if offered.

5. Review the project settings and select **Create build project**.

### Check the CodeBuild project before continuing

Open the project details and verify:

| Item | Expected value |
|---|---|
| Source type | CodePipeline |
| Buildspec | `buildspec.yml` from the source root |
| Artifact type | CodePipeline (not No artifacts) |
| Service role | A CodeBuild service role is assigned |

The project is intended to be invoked by CodePipeline. You do not need to start this project manually from the CodeBuild console.

---

# Part D – Create the CI/CD pipeline

## Step 6. Start creating the pipeline

1. Search for **AWS CodePipeline** and open the service.
2. Select **Create pipeline**.
3. If a creation-options page appears, choose **Build custom pipeline**.
4. Select **Next**.

## Step 7. Configure pipeline settings

Enter the following settings:

| Setting | Value |
|---|---|
| Pipeline name | `Student-CICD-Pipeline` |
| Pipeline type | Keep the console default |
| Service role | New service role |
| Artifact store | Default location |
| Encryption key | Default AWS managed key |

Select **Next**.

The default artifact store is used internally by CodePipeline. It is not the application deployment bucket created in Step 4.

## Step 8. Configure the Source stage

1. For **Source provider**, select **GitHub (via GitHub App)**.
2. Under **Connection**, select an existing available GitHub connection or create a new one.

### If you need to create a GitHub connection

1. Select **Connect to GitHub**.
2. Enter a connection name, for example `Student-GitHub-Connection`.
3. Select **Connect to GitHub** and authorize the AWS Connector for GitHub.
4. Install the AWS Connector application in your GitHub account if prompted.
5. Grant the application access to the `aws-cicd-demo` repository.
6. Return to the AWS console and finish creating the connection.
7. Confirm that the connection status is **Available**, then select it.

For a personal GitHub repository, you must have the required repository-owner permissions to install or authorize the GitHub application. Organization repositories may require an organization owner or administrator.

### Complete the Source settings

| Setting | Value |
|---|---|
| Repository | `aws-cicd-demo` |
| Branch | `main` |
| Output artifact format | **CodePipeline default** |
| Trigger | Keep the default push/change-detection setting |

Choose **CodePipeline default**, not **Full clone**. The default option packages the repository files as a ZIP artifact for the next stage. Full clone requires additional permissions and is not needed for this exercise.

Select **Next**.

## Step 9. Configure the Build stage

1. On the build-stage page, choose **Other build providers** if that option is displayed.
2. For **Build provider**, select **AWS CodeBuild**.
3. For **Project name**, select `Student-CICD-Build`.
4. Select the same Region as the CodeBuild project.
5. If the console asks for the build specification, select **Use a buildspec file**. The filename is `buildspec.yml`.
6. Keep the build type as **Single build**.
7. Continue to the next stage.

The Source stage output is passed to CodeBuild as its input. CodeBuild executes the buildspec and returns a build output artifact to CodePipeline.

### Verify the artifact hand-off

The names shown by the console may be generated automatically. Use the artifact names displayed in your pipeline. The intended mapping is:

| Pipeline stage | Artifact role | Example name |
|---|---|---|
| Source | Output artifact | `SourceArtifact` |
| Build | Input artifact | `SourceArtifact` |
| Build | Output artifact | `BuildArtifact` |
| Deploy | Input artifact | `BuildArtifact` |

The **Deploy input must be exactly the same artifact as the Build output**. If the console generates a different output artifact name, use that exact name in the Deploy stage.

## Step 10. Skip the Test stage

For this introductory exercise, no separate Test stage is required.

1. Select **Skip test stage**, if the wizard displays this option.
2. Confirm the selection if prompted.
3. Continue to the Deploy stage.

## Step 11. Configure the Deploy stage

1. For **Deploy provider**, select **Amazon S3**.
2. Configure the deployment action:

   | Setting | Value |
   |---|---|
   | Region | Same Region as the pipeline and deployment bucket |
   | Bucket | Select the application deployment bucket created in Step 4 |
   | Extract file before deploy | **Enabled / checked** |
   | Deployment path | Leave blank |
   | Canned ACL | Leave blank |
   | Cache control | Leave blank |

3. For the Deploy action's input artifact, select the **Build stage output artifact** (for example, `BuildArtifact`).
4. Do not select `SourceArtifact` if the Build stage has produced a separate build output artifact.
5. Continue to the Review page.

**Why is extraction enabled?** CodePipeline passes the build output as an artifact package. The S3 Deploy action must extract that package so that `index.html` is copied into the deployment bucket as a file, rather than uploaded as a ZIP file.

## Step 12. Review and create the pipeline

Before creating the pipeline, verify:

| Stage | Setting | Expected value |
|---|---|---|
| Source | Provider | GitHub (via GitHub App) |
| Source | Repository / branch | `aws-cicd-demo` / `main` |
| Source | Output artifact | `SourceArtifact` (or console-generated name) |
| Build | Provider | AWS CodeBuild |
| Build | Project | `Student-CICD-Build` |
| Build | Input artifact | Source stage output |
| Build | Output artifact | `BuildArtifact` (or console-generated name) |
| Deploy | Provider | Amazon S3 |
| Deploy | Bucket | Your application deployment bucket |
| Deploy | Input artifact | Build stage output |
| Deploy | Extract file before deploy | Enabled |

Select **Create pipeline**. The first pipeline execution should start automatically.

---

# Part E – Verify the deployment

## Step 13. Check the pipeline execution

Open `Student-CICD-Pipeline` and observe the three stages:

1. **Source:** CodePipeline retrieves the files from the GitHub `main` branch.
2. **Build:** CodeBuild reads `buildspec.yml`, checks `index.html`, copies it into `output`, and packages the file as a build artifact.
3. **Deploy:** The S3 deployment action extracts the build artifact and copies `index.html` into the application deployment bucket.

All three stages should show **Succeeded**.

## Step 14. Inspect the CodeBuild logs

1. In the pipeline view, open the Build action details.
2. Open the CodeBuild build logs in CloudWatch Logs.
3. Look for the build messages:

```text
Starting application build
Application packaged successfully
```

4. Confirm that the build's artifact upload phase completed successfully. The build output must be made available to CodePipeline for the Deploy stage.

## Step 15. Verify the file in Amazon S3

1. Open the Amazon S3 console.
2. Open your application deployment bucket.
3. Confirm that `index.html` is present.
4. Select the file and choose **Download**.
5. Open the downloaded file in a web browser or text editor.

You should see the HTML page containing:

**Welcome to AWS CI/CD!**

> The bucket remains private. This exercise verifies that the application file was deployed to S3; it does not create a public website URL.

---

# Part F – Test automatic deployment

## Step 16. Change the application in GitHub

1. Return to the `aws-cicd-demo` repository in GitHub.
2. Open `index.html` and select Edit.
3. Change this heading:

```html
<h1>Welcome to AWS CI/CD!</h1>
```

To:

```html
<h1>My First Automated AWS Deployment!</h1>
```

4. Select **Commit changes** and commit directly to the `main` branch.

## Step 17. Observe the new pipeline execution

1. Return to AWS CodePipeline.
2. Open `Student-CICD-Pipeline`.
3. Observe the new execution triggered by the GitHub commit.
4. Wait for Source, Build, and Deploy to succeed.
5. Open the deployment bucket in Amazon S3.
6. Download `index.html` again.
7. Confirm that the updated heading appears in the file.

**Result:** A change committed to GitHub automatically triggered the pipeline, CodeBuild generated a new artifact, and the Deploy stage updated the file in Amazon S3.

---

# Troubleshooting

| Symptom | What to check |
|---|---|
| Source stage fails | Check the GitHub connection is **Available**, the AWS Connector has access to the repository, and the correct branch is selected. |
| Build stage fails with a missing buildspec | Confirm the filename is exactly `buildspec.yml` and it is in the repository root. |
| Build stage fails because `index.html` is missing | Confirm `index.html` is in the repository root and committed to `main`. |
| Build succeeds but Deploy reports `NoSuchKey` | Check that CodeBuild produced an output artifact and that the Deploy action input is exactly the Build action output artifact. |
| Build action starts twice for one commit | Check that the CodeBuild project is not also configured with a separate GitHub webhook. Let CodePipeline trigger the build. |
| Deploy reports `AccessDenied` | Check the CodePipeline service role has permission to write to the application deployment bucket. |
| GitHub connection cannot access the repository | Revisit the AWS Connector installation in GitHub and grant access to the correct repository. |

## Important artifact check for a `NoSuchKey` error

If the Deploy stage reports an error similar to `The input artifact ... does not exist` or `NoSuchKey`, check these items in order:

1. Open the Build action details and identify its **Output artifact** name.
2. Confirm that the CodeBuild project artifact type is **CodePipeline**, not **No artifacts**.
3. Confirm that the `buildspec.yml` contains an `artifacts` section and that the file listed there exists after the build commands run.
4. Confirm that the Deploy action uses the exact Build output artifact as its input.
5. Re-run the pipeline after correcting the configuration.

Do not manually upload, rename, or delete objects in the internal `codepipeline-...` artifact bucket. CodePipeline manages those objects.

## If the Deploy stage reports `AccessDenied`

The CodePipeline service role needs permission to deploy objects to the application bucket. An administrator can open **IAM > Roles**, select the service role associated with `Student-CICD-Pipeline`, and add a policy statement scoped to the deployment bucket.

The following is a policy statement example. Replace `<ACCOUNT_ID>` and `<DEPLOYMENT_BUCKET_NAME>` with your own values. Add the statement to the existing role policy; do not replace the role's other required permissions.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowDeployToStudentBucket",
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:PutObjectAcl",
        "s3:PutObjectVersionAcl"
      ],
      "Resource": "arn:aws:s3:::<DEPLOYMENT_BUCKET_NAME>/*",
      "Condition": {
        "StringEquals": {
          "aws:ResourceAccount": "<ACCOUNT_ID>"
        }
      }
    },
    {
      "Sid": "AllowReadDeploymentBucketInformation",
      "Effect": "Allow",
      "Action": [
        "s3:GetBucketVersioning",
        "s3:GetBucketAcl",
        "s3:GetBucketLocation"
      ],
      "Resource": "arn:aws:s3:::<DEPLOYMENT_BUCKET_NAME>",
      "Condition": {
        "StringEquals": {
          "aws:ResourceAccount": "<ACCOUNT_ID>"
        }
      }
    }
  ]
}
```

If a GitHub connection or CodeBuild action reports an IAM permission error, the pipeline service role may also need the relevant `codeconnections:UseConnection` / `codestar-connections:UseConnection` and CodeBuild action permissions. Ask the AWS account administrator to review the role against the AWS documentation links below. Do not add broad `AdministratorAccess` permissions just to make the lab work.

---

# Cleanup – Delete lab resources

To avoid ongoing charges and leave the AWS account tidy:

1. Open CodePipeline and delete `Student-CICD-Pipeline`.
2. Open CodeBuild and delete `Student-CICD-Build`.
3. Open Amazon S3 and empty the application deployment bucket.
4. Delete the application deployment bucket.
5. The pipeline artifact-store bucket may remain. Delete it only if it was created for this lab, no other pipeline uses it, and the pipeline has already been deleted.
6. The GitHub repository and GitHub connection can be retained for future exercises or removed if no longer needed.

---

# Student completion checklist

- [ ] Created the GitHub repository.
- [ ] Added `index.html`.
- [ ] Added `buildspec.yml`.
- [ ] Created the private S3 deployment bucket.
- [ ] Created a CodeBuild project with Source = CodePipeline and Artifacts = CodePipeline.
- [ ] Created a GitHub connection in CodePipeline.
- [ ] Configured the Source stage.
- [ ] Configured the Build stage with `Student-CICD-Build`.
- [ ] Confirmed the Build output artifact is used as the Deploy input artifact.
- [ ] Configured the S3 Deploy stage with extraction enabled.
- [ ] Verified that all pipeline stages succeeded.
- [ ] Downloaded and checked `index.html` from S3.
- [ ] Changed the GitHub file and observed automatic deployment.
- [ ] Cleaned up the AWS resources.

# Key learning points

| AWS service / concept | What you learned |
|---|---|
| GitHub | Stores and versions the application source code. |
| AWS CodePipeline | Orchestrates the Source, Build, and Deploy stages. |
| AWS CodeBuild | Executes build commands from `buildspec.yml`. |
| Build artifact | Packages the files produced by the Build stage for the next stage. |
| Amazon S3 | Stores the deployed application file. |
| CI/CD | Automates the process from a source-code change to deployment. |

## Official AWS documentation

- [Create a pipeline, stages, and actions – AWS CodePipeline](https://docs.aws.amazon.com/codepipeline/latest/userguide/pipelines-create.html)
- [GitHub connections – AWS CodePipeline](https://docs.aws.amazon.com/codepipeline/latest/userguide/connections-github.html)
- [Create a pipeline that uses CodeBuild – AWS CodeBuild](https://docs.aws.amazon.com/codebuild/latest/userguide/how-to-create-pipeline-console.html)
- [Create a build project – AWS CodeBuild](https://docs.aws.amazon.com/codebuild/latest/userguide/create-project.html)
- [Buildspec reference – AWS CodeBuild](https://docs.aws.amazon.com/codebuild/latest/userguide/build-spec-ref.html)
- [Amazon S3 deploy action reference – AWS CodePipeline](https://docs.aws.amazon.com/codepipeline/latest/userguide/action-reference-S3Deploy.html)
- [Manage the CodePipeline service role – AWS CodePipeline](https://docs.aws.amazon.com/codepipeline/latest/userguide/how-to-custom-role.html)

**End of Demo 2**

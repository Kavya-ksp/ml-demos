# Demo 1 – Run Your First Build Using AWS CodeBuild

**Topic:** Building Microservices and a CI/CD Pipeline with AWS  
**Level:** Beginner  
**Duration:** 10–15 minutes  
**Mode:** Individual hands-on exercise – each participant (PAX) follows the instructions  
**AWS service:** AWS CodeBuild  
**Source control:** GitHub

---

## 1. Objective

In this exercise, you will create an AWS CodeBuild project that retrieves source files from a GitHub repository and runs a set of build commands.

By the end of this demo, you will be able to:

1. Store application files in a GitHub repository.
2. Create an AWS CodeBuild project.
3. Configure a build environment and a build specification (`buildspec.yml`).
4. Start a build manually.
5. Review the build status and execution logs.

> **What this demo covers:** This is an introductory build-and-validation exercise. It does not create a deployable output package. In Demo 2, you will configure CodeBuild to produce an output artifact for AWS CodePipeline.

## 2. Architecture

```text
GitHub repository
       |
       | Source files
       v
AWS CodeBuild project
       |
       | Executes buildspec.yml
       v
Build commands and validation
       |
       v
Build status and CloudWatch Logs
```

## 3. Prerequisites

Before you start, make sure you have:

- An AWS account or an instructor-provided AWS lab account.
- Permission to create a CodeBuild project and an IAM service role, and to pass that role to CodeBuild.
- Permission to create or use an AWS CodeConnections GitHub connection if the console asks you to connect GitHub.
- A GitHub account.
- Access to the AWS Management Console.

**Important:** Do not use the AWS account root user for this exercise. Use the student IAM user, role, or lab credentials provided by your instructor. Use the same AWS Region throughout the exercise.

## 4. Part A – Create the GitHub repository

### Step 1: Create a repository

1. Sign in to [GitHub](https://github.com/).
2. Select **New repository**.
3. Enter the following details:

   | Setting | Value |
   |---|---|
   | Repository name | `aws-codebuild-demo` |
   | Visibility | Public |
   | Initialize repository | Add a README file |
   | Default branch | `main` |

4. Select **Create repository**.

A public repository is used to keep source access simple for this beginner exercise. Do not add passwords, access keys, tokens, or other confidential information to the repository.

### Step 2: Create the build specification file

1. In the new repository, select **Add file** > **Create new file**.
2. Enter the filename exactly as shown below:

   `buildspec.yml`

3. Copy the following YAML into the file:

```yaml
version: 0.2

phases:
  build:
    commands:
      - echo "Welcome to AWS CodeBuild"
      - echo "Starting the build"
      - echo "Current source directory:"
      - pwd
      - echo "Files in the repository:"
      - ls -la
      - test -f README.md
      - echo "README.md was found"
      - echo "Build completed successfully"
```

4. Select **Commit changes**.
5. Commit the file directly to the `main` branch.

### Step 3: Verify the repository structure

Your repository should contain at least these files:

```text
aws-codebuild-demo/
|
|-- README.md
`-- buildspec.yml
```

**What is `buildspec.yml`?**  
A buildspec is a YAML file that tells CodeBuild which commands to execute and in which build phase to execute them. CodeBuild looks for a file named `buildspec.yml` in the root of the source directory by default.

The commands in this exercise print a welcome message, display the working directory and repository files, and check that `README.md` exists. The `test -f README.md` command returns a failure status if the file is missing, which makes it a simple validation check.

## 5. Part B – Create an AWS CodeBuild project

### Step 4: Open AWS CodeBuild

1. Sign in to the [AWS Management Console](https://console.aws.amazon.com/).
2. Use the Region selector in the top navigation bar to select the Region assigned by your instructor.
3. Search for **CodeBuild** and open **AWS CodeBuild**.
4. In the navigation pane, select **Build projects**.
5. Select **Create build project**.

### Step 5: Configure the project

Under **Project configuration**, enter:

| Setting | Value |
|---|---|
| Project name | `Student-Build-Demo` |
| Description | `First AWS CodeBuild project for learning CI` |

The project name must be unique within your AWS account. If it is already in use, add your initials or another short identifier.

### Step 6: Configure the source

Under **Source**:

| Setting | Value |
|---|---|
| Source provider | GitHub |
| Repository | Public repository |
| Repository URL | HTTPS URL of your `aws-codebuild-demo` repository |
| Source version | `main` |

Example repository URL format:

```text
https://github.com/<your-github-username>/aws-codebuild-demo
```

If the console displays GitHub credential or connection settings, choose the supported GitHub App connection option and create or select a GitHub connection. Follow the on-screen instructions to authorize AWS and grant the connection access to the repository. Then return to CodeBuild and select the repository. Do not paste a personal access token into the buildspec or source URL.

For this exercise, **do not configure a webhook trigger**. You will start the build manually from the CodeBuild console. If a section named **Primary source webhook events** appears, leave it unconfigured.

### Step 7: Configure the build environment

Under **Environment**, use the following beginner-friendly settings. Some labels can vary slightly as AWS updates the console.

| Setting | Value |
|---|---|
| Provisioning model | On-demand |
| Environment image | Managed image |
| Compute | EC2 |
| Operating system | Ubuntu |
| Runtime | Standard |
| Image | Latest supported Standard image offered in the console |
| Image version | Latest available version for the selected image |
| Compute type | Small |
| Service role | New service role |

If the console asks for an environment type or architecture, choose the standard Linux x86_64 option. Do not enable **Privileged** mode; this exercise does not build Docker images. Do not configure a VPC, environment variables, or a cache.

**Service role:** Select **New service role** and allow CodeBuild to create the role. This role allows the CodeBuild build environment to access the AWS resources it needs, including writing build logs to CloudWatch Logs. The person creating the project must have permission to create the role and pass it to CodeBuild. If your instructor has provided an approved existing CodeBuild service role, use that role instead.

### Step 8: Configure the buildspec

Under **Buildspec**:

1. Select **Use a buildspec file**.
2. Set the buildspec filename to `buildspec.yml` if the console asks for a filename.
3. Keep the file in the root of the GitHub repository.

Do not select **Insert build commands** for this exercise because the commands are already stored in the repository's `buildspec.yml` file.

### Step 9: Configure artifacts

Under **Artifacts**:

- Select **No artifacts**.

This is intentional. The purpose of Demo 1 is to run commands and validate the source files. We are not creating a ZIP package or sending output to S3 in this demo.

> **Important distinction:** `No artifacts` is correct for this Demo 1 exercise. Do not copy this artifact setting into Demo 2. The CodeBuild project used by CodePipeline in Demo 2 must use the CodePipeline artifact integration and produce an output artifact for the Deploy stage.

### Step 10: Configure logs

Under **Logs**:

1. Enable **CloudWatch Logs**.
2. Keep the default log group and stream settings, or use the names suggested by the console.
3. Leave Amazon S3 logs disabled; they are not required for this exercise.

CloudWatch Logs allows you to inspect the commands and their output after a build runs.

### Step 11: Create the project

1. Review the project configuration.
2. Confirm that the source is your GitHub repository, the buildspec is `buildspec.yml`, and the artifact type is **No artifacts**.
3. Select **Create build project**.

You should now see `Student-Build-Demo` in the list of CodeBuild projects.

## 6. Part C – Run the build

### Step 12: Start a build

1. Open **AWS CodeBuild** > **Build projects**.
2. Select `Student-Build-Demo`.
3. Select **Start build**.
4. Keep the default build settings and select **Start build** again if the console asks you to confirm.

CodeBuild provisions the build environment, downloads the source from GitHub, reads `buildspec.yml`, and runs the commands in the build phase.

### Step 13: Monitor the build

Open the build execution details. Wait until the build reaches a final status.

Expected result:

**Build status: `Succeeded`**

Open the build logs or the CloudWatch Logs link. You should find output similar to:

```text
Welcome to AWS CodeBuild
Starting the build
Current source directory:
[working directory shown by the build environment]
Files in the repository:
README.md
buildspec.yml
README.md was found
Build completed successfully
```

The exact working directory and file listing can differ. The important checks are that the repository files were downloaded, the `README.md` check passed, and the build status is **Succeeded**.

### Step 14: Understand the build execution

| Item | What it does |
|---|---|
| GitHub | Stores the source files and buildspec file |
| AWS CodeBuild | Provides the managed environment and runs the build commands |
| `buildspec.yml` | Defines the build phases and commands |
| IAM service role | Allows CodeBuild to access required AWS resources |
| Amazon CloudWatch Logs | Stores the build output and execution logs |

**Instructor explanation:**

> A developer stores application code in GitHub. AWS CodeBuild retrieves that source code and executes the instructions in `buildspec.yml`. If all commands finish successfully, CodeBuild reports that the build succeeded. This is a basic building block of Continuous Integration.

## 7. Optional exercise – Change the source and run the build again

1. Open the GitHub repository.
2. Edit `README.md` and add this line:

   `This is my second CodeBuild execution.`

3. Commit the change to `main`.
4. Return to AWS CodeBuild.
5. Open `Student-Build-Demo` and select **Start build**.
6. Review the new build logs and confirm that the build succeeds.

Because this lab does not configure a webhook, a GitHub commit does **not** start a build automatically. You start each build manually. Automatic source-change triggering will be explored in the CI/CD pipeline exercise.

## 8. Troubleshooting

| Issue | What to check |
|---|---|
| Source download fails | Confirm the repository URL is correct, the repository exists, and the selected branch is `main`. Check the GitHub connection if one was configured. |
| Buildspec file not found | Confirm the file is named exactly `buildspec.yml` and is in the repository root. Check that it was committed to `main`. |
| YAML or buildspec error | Check indentation. YAML uses spaces; do not use tab characters. Compare the file with the example in Step 2. |
| `README.md` check fails | Confirm `README.md` exists in the repository root and is committed to the selected branch. |
| Build cannot create or use the service role | Ask the instructor or AWS administrator to verify the student's IAM permissions, including permission to create and pass the CodeBuild service role. |
| CloudWatch logs are unavailable | Confirm CloudWatch Logs is enabled in the project and the CodeBuild service role has the required log permissions. |
| Build status is `Failed` | Open the build logs, find the first command that failed, correct the cause, and start the build again. |

## 9. Cleanup

To avoid leaving unused resources in the student AWS account:

1. Open **AWS CodeBuild** > **Build projects**.
2. Select `Student-Build-Demo` and delete the build project.
3. If a CloudWatch log group was created specifically for this exercise (commonly named `/aws/codebuild/Student-Build-Demo`), delete it only after you no longer need the logs.
4. Delete the CodeBuild IAM service role only if it was created exclusively for this exercise and is not used by another project. Do not delete shared or instructor-provided roles.
5. Keep a GitHub repository only if you plan to reuse it for Demo 2; otherwise, delete it from GitHub when finished.
6. Keep a GitHub connection if it will be reused in Demo 2 or by another project. Do not delete a shared connection.

CodeBuild and CloudWatch Logs usage may incur charges depending on the account, Region, build environment, and usage. Check the applicable AWS pricing and your lab-account rules. Delete resources after the exercise.

## 10. Student completion checklist

- [ ] Created the public GitHub repository `aws-codebuild-demo`.
- [ ] Added `buildspec.yml` to the repository root.
- [ ] Created the `Student-Build-Demo` CodeBuild project.
- [ ] Configured GitHub as the source provider and selected the `main` branch.
- [ ] Selected a managed Linux Standard build environment.
- [ ] Selected a CodeBuild service role.
- [ ] Selected the repository's `buildspec.yml` file.
- [ ] Selected **No artifacts** for this introductory build-only exercise.
- [ ] Enabled CloudWatch Logs.
- [ ] Started the build manually.
- [ ] Verified the build status is **Succeeded** and inspected the logs.

## 11. Official AWS references

- [Create a build project in AWS CodeBuild](https://docs.aws.amazon.com/codebuild/latest/userguide/create-project.html)
- [AWS CodeBuild buildspec reference](https://docs.aws.amazon.com/codebuild/latest/userguide/build-spec-ref.html)
- [Getting started with AWS CodeBuild](https://docs.aws.amazon.com/codebuild/latest/userguide/getting-started-overview.html)
- [AWS CodeBuild pricing](https://aws.amazon.com/codebuild/pricing/)

---

**End of Demo 1 – Run Your First Build Using AWS CodeBuild**

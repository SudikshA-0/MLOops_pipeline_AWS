# Iris ML API on AWS

This repository deploys the Iris FastAPI service on AWS. GitHub is the source repository; AWS CodePipeline runs the pipeline, CodeBuild runs the tests and builds/pushes the image to Amazon ECR, and CloudFormation deploys it as an Amazon ECS Express Mode service.

The service uses the checked-in model at `models/iris_random_forest_model.joblib`, so it does not need an MLflow server to start. To use MLflow instead, configure `MLFLOW_MODEL_URI` and `MLFLOW_TRACKING_URI` on the service. Model training is not part of this deployment pipeline.

## Deploy from the AWS console

Use the **Asia Pacific (Mumbai) `ap-south-1`** region for this walkthrough. You need an AWS identity allowed to create CloudFormation stacks, IAM roles, CodePipeline, CodeBuild, ECR, ECS, and a GitHub connection. Do not use your AWS root user for routine work.

### 1. Create the AWS pipeline stack

1. In GitHub, open this repository and download `infra/aws-pipeline.yaml` (open the file, then use **Download raw file**).
2. Sign in to the [AWS Management Console](https://console.aws.amazon.com/) and select **Asia Pacific (Mumbai) `ap-south-1`** in the region selector.
3. Open **CloudFormation** and choose **Create stack → With new resources (standard)**.
4. Under **Prepare template**, choose **Choose an existing template**. Under **Specify template**, choose **Upload a template file**, select the downloaded `aws-pipeline.yaml`, and choose **Next**.
5. Set the stack name to `iris-mlops-aws-pipeline`. Keep the defaults, especially owner `SudikshA-0`, repository `MLOops_pipeline_AWS`, and branch `main`. Choose **Next**.
6. On **Configure stack options**, choose **Next**. Review the changes, check the acknowledgement that CloudFormation may create IAM resources, and choose **Submit**.
7. Wait for the stack status to become **CREATE_COMPLETE**. In **Outputs**, note the GitHub connection ARN and pipeline name.

### 2. Link AWS to GitHub

1. In the same AWS region, open **Developer Tools → Settings → Connections** (the console may label this **CodeConnections**).
2. Find the connection named `iris-mlops-github`. Its status starts as **Pending** because CloudFormation cannot authorize your GitHub account for you.
3. Open the connection and choose **Update pending connection** or **Connect to GitHub**.
4. Choose **Connect to GitHub** and then **Authorize AWS Connector for GitHub**.
5. Install or configure the AWS Connector for GitHub. Grant it access to `MLOops_pipeline_AWS`, then choose **Install**, **Connect**, or **Save** as shown by GitHub and AWS.
6. Return to the AWS Connections page and wait until the connection status is **Available**.

### 3. Run and inspect the deployment

1. Open **CodePipeline**, choose `iris-mlops-aws-pipeline`, and choose **Release change** if the first run has not started by itself.
2. Follow the **Source**, **Build**, and **Deploy** stages. The Build stage runs `pytest`, builds the Docker image, and pushes it to ECR. The Deploy stage creates or updates the `iris-mlops-app` CloudFormation stack.
3. When all stages succeed, open **CloudFormation → Stacks → iris-mlops-app → Outputs** and copy `ServiceEndpoint`.
4. Open `<ServiceEndpoint>/health` to check health, or `<ServiceEndpoint>/docs` to try the API. The prediction endpoint is `POST <ServiceEndpoint>/predict`.

After this first successful run, pushes to `main` in this GitHub repository start the AWS pipeline automatically.

## What this creates and cost

The pipeline stack creates a CodePipeline, CodeBuild project, ECR repository, GitHub connection, S3 artifact bucket, and IAM roles. The app stack creates an ECS Express Mode service. The running service uses Fargate and managed load-balancing resources; these and build minutes, logs, image storage, and S3 storage can incur AWS charges. Review the AWS pricing pages for your selected region and monitor **Billing and Cost Management**.

## Remove the resources

1. In CloudFormation, delete `iris-mlops-app` first.
2. Delete `iris-mlops-aws-pipeline` next.
3. The ECR repository and artifact bucket are retained to protect stored data. If you no longer need them, empty the S3 artifact bucket, delete it, then delete the ECR images and repository from the ECR console.

## Run locally

Install dependencies with `pip install -r requirements.txt`, then start the API with `uvicorn main:app --host 0.0.0.0 --port 8000`. Local startup loads the bundled model by default. The API docs are at `/docs`.

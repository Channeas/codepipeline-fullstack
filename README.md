# CodePipeline fullstack CICD pipeline

This is a [CloudFormation](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/Welcome.html) template that creates a [CodePipeline](https://aws.amazon.com/codepipeline/) pipeline that builds and deploys fullstack applications on AWS. It builds git projects, that are fetched using a [CodeStar Connection](https://docs.aws.amazon.com/codestar-connections/latest/APIReference/Welcome.html). The frontend is deployed on a [CloudFront](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/Introduction.html) distribution.

## Resources created

The template creates a total of 8 resources, which are listed below:

-   3 [S3](https://aws.amazon.com/s3/) buckets - 1 for storing artifacts used by the CodePipeline pipeline, 1 for storing the frontend after building it, and 1 for storing backend code (for example zipped Lambdas)

-   2 [CodeBuild](https://aws.amazon.com/codebuild/) builders. One is used to build the frontend, and the other one to build the backend

-   1 [CodePipeline](https://aws.amazon.com/codepipeline/) pipeline that retrieves the source code from the git provider, and then builds the frontend and backend

-   1 [CloudFront](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/Introduction.html) distribution that serves the frontend, as well as an [Origin Access Identity](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/private-content-restricting-access-to-s3.html) used by the distribution to retrieve content

-   1 (nested) [CloudFormation](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/Welcome.html) stack that contains the backend resources speicifed by the backend template

The template also creates 3 [IAM roles](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles.html) and 1 [S3 bucket policy](https://docs.aws.amazon.com/AmazonS3/latest/dev/using-iam-policies.html):

-   1 role for the [CodePipeline](https://aws.amazon.com/codepipeline/) pipeline that allows access to the specified [CodeStar Connection](https://docs.aws.amazon.com/codestar-connections/latest/APIReference/Welcome.html), the 3 [S3](https://aws.amazon.com/s3/) buckets, and the [CodeBuild](https://aws.amazon.com/codebuild/) builder

-   1 role for the [CodeBuild](https://aws.amazon.com/codebuild/) builders that allows

    -   Access to [CloudWatch](https://aws.amazon.com/cloudwatch/) and [CloudWatch Logs](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/WhatIsCloudWatchLogs.html) for storing build logs
    -   [Read](https://docs.aws.amazon.com/AmazonS3/latest/API/API_GetObject.html) and [write](https://docs.aws.amazon.com/AmazonS3/latest/API/API_PutObject.html) access to the artifact [S3](https://aws.amazon.com/s3/) bucket
    -   [ListBucket](https://docs.aws.amazon.com/AmazonS3/latest/dev/walkthrough1.html#walkthrough-group-policy:~:text=List%20root%2Dlevel%20items%2C%20folders%2C%20and%20objects,have%20permission%20for%20the%20s3%3AListBucket%20action) and [DeleteObject](https://docs.aws.amazon.com/AmazonS3/latest/API/API_DeleteObject.html) access to the frontend and backend [S3](https://aws.amazon.com/s3/) buckets for clearing legacy files

-   1 role that determines what resources the backend can contain (called "ChangeSetRole")

-   1 bucket policy granting the [CloudFront](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/Introduction.html) distribution [read](https://docs.aws.amazon.com/AmazonS3/latest/API/API_GetObject.html) access to the frontend bucket

## Pipeline architecture

The pipeline consists of 6 stages, shown in the diagram below:

![Diagram of how the pipeline is structured](architecture.png)

The stages do the following:

1. **Source** - retrieves the source code using the [CodeStar Connection](https://docs.aws.amazon.com/codestar-connections/latest/APIReference/Welcome.html) specified as the _CodeStarConnection_ parameter
2. **Build-and-zip** - uses 2 [CodeBuild](https://aws.amazon.com/codebuild/) builders to build the frontend and zip backend files
3. **Deploy-to-S3** - deploys the files from the builders to [S3](https://aws.amazon.com/s3/). The frontend goes to a bucket used by the [CloudFront](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/Introduction.html) distribution, while the backend goes to another bucket for temporary storage
4. **Create-backend-changeset** - creates a [CloudFormation](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/Welcome.html) changeset identifying what has changed since the backend template was last deployed
5. **Approve-backend-changeset** - [manual approval action](https://docs.aws.amazon.com/codepipeline/latest/userguide/approvals.html) of the backend changes, with a notification sent to the SNS topic specified as the _ApprovalSNSTopicARN_ parameter
6. **Execute-backend-changeset** - assuming the backend changes were approved, this stage builds the new backend resources and modifies existing ones

It is worth noting that unlike [Lambda](https://docs.aws.amazon.com/lambda/index.html) functions specified in a regular [CloudFormation](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/Welcome.html) template, the ones specified in the backend template will automatically update each time the pipeline runs. This is achieved by storing the backend builds in new folders, leading to updated [S3](https://aws.amazon.com/s3/) keys for the [Lambdas](https://docs.aws.amazon.com/lambda/index.html)

## Required IAM permissions

## Template parameters

## Setup

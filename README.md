# CodePipeline fullstack CICD pipeline

This is a [CloudFormation](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/Welcome.html) template that creates a [CodePipeline](https://aws.amazon.com/codepipeline/) pipeline that builds and deploys fullstack applications on AWS. It builds git projects, that are fetched using a [CodeStar Connection](https://docs.aws.amazon.com/codestar-connections/latest/APIReference/Welcome.html). The frontend is deployed on a [CloudFront](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/Introduction.html) distribution.

## Resources created

The template creates a total of 8 resources, which are listed below:

-   3 [S3](https://aws.amazon.com/s3/) buckets - 1 for storing artifacts used by the CodePipeline pipeline, 1 for storing the frontend after building it, and 1 for storing backend code (for example zipped Lambdas)

-   2 [CodeBuild](https://aws.amazon.com/codebuild/) builders. One is used to build the frontend, and the other one to build the backend

-   1 [CodePipeline](https://aws.amazon.com/codepipeline/) pipeline that retrieves the source code from the git provider, and then builds the frontend and backend

-   1 [CloudFront](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/Introduction.html) distribution that serves the frontend, as well as an [Origin Access Identity](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/private-content-restricting-access-to-s3.html) used by the distribution to retrieve content

-   1 (nested) [CloudFormation](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/Welcome.html) stack that contains the backend resources speicifed by the backend template

The template also creates 3 [IAM roles](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles.html) and 1 [S3 bucket policy](https://docs.aws.amazon.com/AmazonS3/latest/dev/using-iam-policies.html), which are described under the "Required IAM permissions" header.

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

To create the template, an IAM user requires the following permissions:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Action": [
                "cloudformation:CreateStack",
                "cloudformation:CreateChangeSet",
                "cloudformation:DeleteChangeSet",
                "cloudformation:DescribeChangeSet",
                "cloudformation:DescribeStacks",
                "cloudformation:DescribeStackEvents",
                "cloudformation:DescribeStackResources",
                "cloudformation:ExecuteChangeSet"
                "cloudformation:GetTemplate",
                "cloudformation:ValidateTemplate",
            ],
            "Resource": "*",
            "Effect": "Allow"
        },
        {
            "Action": [
                "codebuild:*",
                "codepipeline:*",
                "cloudfront:*",
                "s3:*",
                "iam:*"
            ],
            "Resource": "*",
            "Effect": "Allow"
        },
        {
            "Action": ["apigateway:*", "lambda:*", "dynamodb:*"],
            "Resource": "*",
            "Effect": "Allow"
        }
    ]
}

```

Please note that the last statement is only required if you plan to run the [example repository](https://github.com/Channeas/cicd-fullstack-test), and could be omitted otherwise.

The template creates 1 [S3 bucket policy](https://docs.aws.amazon.com/AmazonS3/latest/dev/using-iam-policies.html) 3 [IAM roles](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles.html). Therefore, the permissions below are also required for the IAM user using this template:

### Bucket policy

The following [S3 bucket policy](https://docs.aws.amazon.com/AmazonS3/latest/dev/using-iam-policies.html) is attatched to the frontend [S3](https://aws.amazon.com/s3/) bucket created by the template, granting [CloudFront](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/Introduction.html) read access:

```json
{
    "Version": "2012-10-17",
    "Id": "[ProjectName]-frontend_policy",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::cloudfront:user/CloudFront Origin Access Identity [FrontendOriginAccessIdentity]"
            },
            "Action": "s3:GetObject",
            "Resource": "[FrontendBucket]/*"
        }
    ]
}
```

### Pipeline role

The following IAM role is used by the pipeline. It allows:

-   Permission to use the specified [CodeStar Connection](https://docs.aws.amazon.com/codestar-connections/latest/APIReference/Welcome.html)
-   Full access to the 3 [S3](https://aws.amazon.com/s3/) buckets created by the template
-   Permission to start and access builds for the 2 [CodeBuild](https://aws.amazon.com/codebuild/) builders
-   Permission to publish to the specified [SNS topic](https://aws.amazon.com/sns/)
-   Access to the backend [CloudFormation](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/Welcome.html) stack created by the template, and permission to work with changesets for that stack
-   Permission to pass this role on to the role creating the backend [CloudFormation](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/Welcome.html) changesets

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Action": "codestar-connections:UseConnection",
            "Resource": "[CodeStarConnection]",
            "Effect": "Allow"
        },
        {
            "Action": "s3:*",
            "Resource": [
                "[ArtifactBucket]/*",
                "[BackendBucket]/*",
                "[FrontendBucket]/*"
            ],
            "Effect": "Allow"
        },
        {
            "Action": [
                "codebuild:StartBuild",
                "codebuild:StartBuildBatch",
                "codebuild:BatchGetBuilds",
                "codebuild:BatchGetBuildBatches"
            ],
            "Resource": ["[BackendBuilder]", "[FrontendBuilder]"],
            "Effect": "Allow"
        },
        {
            "Action": "sns:Publish",
            "Resource": "[ApprovalSNSTopicARN]",
            "Effect": "Allow"
        },
        {
            "Action": [
                "cloudformation:CreateChangeSet",
                "cloudformation:DeleteChangeSet",
                "cloudformation:DescribeChangeSet",
                "cloudformation:DescribeStacks",
                "cloudformation:ExecuteChangeSet"
            ],
            "Resource": "[BackendStack]",
            "Effect": "Allow"
        },
        {
            "Action": "iam:PassRole",
            "Resource": "[ChangeSetRole]",
            "Effect": "Allow"
        }
    ]
}
```

### Build role

The following role is used by the 2 [CodeBuild](https://aws.amazon.com/codebuild/). It allows:

-   Access to [CloudWatch](https://aws.amazon.com/cloudwatch/) and [CloudWatch Logs](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/WhatIsCloudWatchLogs.html) for storing build logs
-   [Read](https://docs.aws.amazon.com/AmazonS3/latest/API/API_GetObject.html) and [write](https://docs.aws.amazon.com/AmazonS3/latest/API/API_PutObject.html) access to the artifact [S3](https://aws.amazon.com/s3/) bucket
-   List and object deletion access to the frontend [S3](https://aws.amazon.com/s3/) bucket (for clearing legacy build files)

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Action": [
                "cloudwatch:*",
                "logs:CreateLogGroup",
                "logs:CreateLogStream",
                "logs:PutLogEvents"
            ],
            "Resource": "*",
            "Effect": "Allow"
        },
        {
            "Action": ["s3:PutObject", "s3:GetObject", "s3:GetObjectVersion"],
            "Resource": "[ArtifactBucket]/*",
            "Effect": "Allow"
        },
        {
            "Action": "s3:ListBucket",
            "Resource": "[FrontendBucket]",
            "Effect": "Allow"
        },
        {
            "Action": "s3:DeleteObject",
            "Resource": "[FrontendBucket]/*",
            "Effect": "Allow"
        }
    ]
}
```

### Backend role

The last role created by the template is used to create the changeset that describes changes to the backend [CloudFormation](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/Welcome.html) stack.

The first two statements below are required. The third one, however, is what decides what controls the permissions of the backend. The permissions below are just an example of the permissions that could be used for a serverless app, but they should be changed to fit your backend. This change should ideally be done directly in the template to avoid issues with future template updates, but could be done using the [IAM](https://aws.amazon.com/iam/) console.

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Action": [
                "iam:AttachRolePolicy",
                "iam:CreateRole",
                "iam:DeleteRole",
                "iam:DeleteRolePolicy",
                "iam:DetachRolePolicy",
                "iam:GetRole",
                "iam:getRolePolicy",
                "iam:PassRole",
                "iam:PutRolePolicy",
                "iam:TagRole"
            ],
            "Resource": "*",
            "Effect": "Allow"
        },
        {
            "Action": ["s3:GetObject"],
            "Resource": "[BackendBucket]/*",
            "Effect": "Allow"
        },
        {
            "Action": ["apigateway:*", "lambda:*", "dynamodb:*"],
            "Resource": "*",
            "Effect": "Allow"
        }
    ]
}
```

## Template parameters

## Setup

# CodePipeline fullstack CICD pipeline

This is a [CloudFormation](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/Welcome.html) template that creates a [CodePipeline](https://aws.amazon.com/codepipeline/) pipeline that builds and deploys fullstack applications on AWS. It builds git projects, that are fetched using a [CodeStar Connection](https://docs.aws.amazon.com/codestar-connections/latest/APIReference/Welcome.html). The frontend is deployed on a [CloudFront](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/Introduction.html) distribution.

## Resources created

The template created a total of 7 resources, which are listed below:

-   3 [S3](https://aws.amazon.com/s3/) Buckets - 1 for storing artifacts used by the CodePipeline pipeline, one for storing the frontend after building it, and one for storing backend code (for example zipped Lambdas)

-   1 [CodePipeline](https://aws.amazon.com/codepipeline/) pipeline that retrieves the source code from the git provider, and then builds the frontend and backend

-   1 [CodeBuild](https://aws.amazon.com/codebuild/) builder that is used in the pipeline to build the frontend and backend

-   1 [CloudFront](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/Introduction.html) distribution that serves the frontend

-   1 (nested) [CloudFormation](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/Welcome.html) stack. This stack will contain any resources that are specified by the backend template

## Pipeline architecture

![Diagram of how the pipeline is structured](architecture.png)

## Template parameters

## Setup

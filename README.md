# CodePipeline fullstack CICD pipeline

This is a [CloudFormation](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/Welcome.html) template that creates a [CodePipeline](https://aws.amazon.com/codepipeline/) pipeline that builds and deploys fullstack applications on AWS. It builds git projects, that are fetched using a [CodeStar Connection](https://docs.aws.amazon.com/codestar-connections/latest/APIReference/Welcome.html). The frontend is deployed on a [CloudFront](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/Introduction.html) distribution.

## Resources created

## Pipeline architecture

![Diagram of how the pipeline is structured](architecture.png)

## Template parameters

## Setup

# SAR-Lambda-Janitor

[![Version](https://img.shields.io/badge/semver-1.7.0-blue)](template.yml)
[![Greenkeeper badge](https://badges.greenkeeper.io/lumigo/SAR-Lambda-Janitor.svg)](https://greenkeeper.io/)
[![CircleCI](https://circleci.com/gh/lumigo-io/SAR-Lambda-Janitor.svg?style=svg)](https://circleci.com/gh/lumigo/SAR-Lambda-Janitor)
[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](LICENSE)

Cron job for deleting old, unused versions of your Function.

This [post](https://lumigo.io/blog/a-serverless-application-to-clean-up-old-deployment-packages/) explains the problem and why we created this app.

## Safeguards

To guard against deleting live versions, some safeguards are in place:

* **Never delete the $LATEST version**. This is the default version that will be used when you invoke a function.
* **Never delete versions that are  referenced by an alias**. If you use aliases to manage different stages - dev, staging, etc. then the latest version referenced by your aliases will not be deleted.
* **Keeping the most recent N versions**. Even if you don't use aliases at all, we will always keep the most recent N versions, where N can be configured with the `VersionsToKeep` parameter when you install the app. **Defaults to 3**.

## Deploying to your account (via the console)

Go to this [page](https://serverlessrepo.aws.amazon.com/applications/arn:aws:serverlessrepo:us-east-1:374852340823:applications~lambda-janitor) and click the `Deploy` button.

This app would deploy the following resources to your region:

* a Lambda function that scans the functions in your region and deletes unused versions
* a CloudWatch event schedule that triggers the Lambda function every hour

## Deploying via SAM/Serverless framework/CloudFormation

To deploy this app via SAM, you need something like this in the CloudFormation template:

```yml
AutoDeployMyAwesomeLambdaLayer:
  Type: AWS::Serverless::Application
  Properties:
    Location:
      ApplicationId: arn:aws:serverlessrepo:us-east-1:374852340823:applications/lambda-janitor
      SemanticVersion: <enter latest version>
    Parameters:
      VersionsToKeep: <defaults to 3>
```

To do the same via CloudFormation or the Serverless framework, you need to first add the following `Transform`:

```yml
Transform: AWS::Serverless-2016-10-31
```

For more details, read this [post](https://theburningmonk.com/2019/05/how-to-include-serverless-repository-apps-in-serverless-yml/).

## Why have we forked this repo?
Because we are packaging and publishing it as a private application instead of using their publicly available one. We deploy an instance of this lambda to each service.

## Bellroy process for deploying and testing this

You should test in Playbell account first. You can use the same commands as described below but amended for the Playbell account. Ensure you have an S3 bucket available. You will also have to create lambda functions to be cleaned as well (more than the number that will be retained upon cleaning). When you 'deploy' the Serverless Application, part of the infrastructure it creates is the lambda function that cleans up the existing lambda functions (note, you need to select the checkbox 'Show apps that create custom IAM roles or resource policies' to see the lambda janitor in `private applications` ). You can click the 'test' option to run this lambda function outside of its schedule.

### Steps for production deployment
Note: Serverless applications are present in the shared-msk-vpc account. There is a specific role `serverlessrepo_deploy_role` that will need to be used in the CLI for the following commands, as well as in the console to view the serverless applications.

1. Ensure you have bumped the SemanticVersion number in template.yml to the next version
2. In this repo, run (with profile name matching your .aws/config file):
  ```
  AWS_PROFILE=serverlessrepo_deploy sam package --output-template-file packaged.yaml --s3-bucket sar-lambda-janitor-ls6qh6jc5vra --region us-east-1
  ```
  This creates binaries in the S3 bucket and generates the packaged.yaml file which describes the serverless application
3. Then run:
  ```
  AWS_PROFILE=serverlessrepo_deploy sam publish --template packaged.yaml --region us-east-1
  ```
  To publish to the Serverless Application Repository
4. Repeat the above 2 steps but for the aws-east-2 region
5. Check that the Application at your new version is available in the Serverless Application Repository in both regions
6. In the tf-modules repository, update the variable for the semantic_version to what you have just deployed `tf-modules/terraform-aws-serverless-service_bell-github/modules/sam_cfn_stack/variables.tf` - release the new module version
7. Deploy the new module version to project-evaluator or one low risk account to test. Leave it for a day or so and confirm it is working as expected before deploying wider
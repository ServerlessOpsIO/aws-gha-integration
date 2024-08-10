# aws-gha-integration

AWS and GitHub Actions Integration

This repository provides the necessary AWS IAM roles to allow GitHuib Actions to assume permissions in an AWS account to build and deploy an application using CloudFormation and Cloudformation based tools. (eg. AWS SAM) With this deployed a GitHub Actions workflow can then assume a pre-defined role in target AWS accounts to perform operations using the AWS API.

## Account Types
There is configuration for two account types: Build and Deploy. A Build account may be, and in this case is, also a Deploy account.

### Build Account
Used for resources needed during GitHub Actions build worklows or to store artifacts required to be shared across an application's environments. Nothing runsNothing actually "runs" in this account.

This account provides IAM OIDC configuration and an IAM role GitHub Actions can assume to perform AWS API operations. It provides a central entrypoint from GitHub Actions into all accounts.

It also provides storage of artifacts that need to be shared across environments such as AWS Lambda function artifacts. This ensures that a single build artifact is used across all environments while not requiring application environment accounts to have access to one another. In the case of AWS SAM and Lambda functions they should be stored in this account's shared SAM S3 bucket using the `sam package` command. This way when deploying to eeach environment of an application the same function artifact will be dewployed to each environment.

### Deploy Account
This is any account configured for deployments. Access to this account by GitHub Actions is only possible by assuming a role in the Build account and then assuming a role in the Deploy account. For added security the role assumd in the Deplop account has limited permissions. The role provides enough 


A build account can also be a deploy account.

## Deployment
Installation in accounts is managed via AWS CloudFormation StackSets. A StackSet for both Build and Deploy accounts is deployed to the AWS organization management account. Each StackSet is configured to deploy appropriately across the organization. These StackSets are deployed via a GitHub Actions workflow.

## Authorization Flow
To Build
* The workflow via the [configure-aws-credentials GitHub action](https://github.com/aws-actions/configure-aws-credentials) requests the _GitHubActionsBuildRole_ IAM role in the build account.
    * This role is restricted to GitHub Actions OIDC credentials AND this organization's GitHub organization.
    * A build workflow need only go this far
* This role has limited permissions and scoped to allow the minimum possible AWS API access.

To Deploy:
* The workflow via the [configure-aws-credentials GitHub action](https://github.com/aws-actions/configure-aws-credentials) requests the _GitHubActionsBuildRole_ IAM role in the build account.
* The workflow then requests the _GitHubActionsDeployRole_ IAM role in the target deploy account.
* The _GitHubActionsDeployRole_ IAM role has limited permissions and scoped to allow the minimum possible AWS API access as as much as possible is expected to be done through Cloudformation.
* The workflow should call CloudFormation and pass the _CfnExecIamRole_
    * This role as the account Administrator policy which allows CFN to perform any action.
    * This approach always ensures their is an audit trail of actions
* If there are shared artifacts in an S3 bucket (eg. Lambda function artifacts) in the Build account a bucket policy allows read access to objects by the _GitHubActionsDeployRole_ IAM role from any account within the AWS organization

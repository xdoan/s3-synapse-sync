# s3-synapse-sync

Lambda function code to index files in S3 buckets by creating filehandles on Synapse, triggered by file changes to S3.


## Getting started

Confirm [center onboarding](https://docs.google.com/document/d/1cCRkfK6or6lwMNc96f5LTp9rK8aHEDqhyP0ISgSn9sI/edit) steps are complete, and a Synapse project has been created to which the bucket will be synced.

---

## Development

### Requirements
Run `pipenv install --dev` to install both production and development
requirements, and `pipenv shell` to activate the virtual environment. For more
information see the [pipenv docs](https://pipenv.pypa.io/en/latest/).

After activating the virtual environment, run `pre-commit install` to install
the [pre-commit](https://pre-commit.com/) git hook.

#### Parameters
Create an AWS KMS key to encrypt secure strings.

Create a sceptre s3-synapse-sync-kms-key.yaml file used to deploy cloudformation
template [s3-synapse-sync-kms-key.yaml](s3-synapse-sync-kms-key.yaml):
```yaml
template_path: "s3-synapse-sync-kms-key.yaml"
stack_name: "s3-synapse-sync-kms-key"
stack_tags:
  Department: "CompOnc"
  Project: "HTAN"
  OwnerEmail: "joe.smith@sagebase.org"
```
__Note__: You may need to add your user ARN to the policy principal in the
cloudformation template.

Deploy the stack using sceptre:
```shell script
sceptre --var "profile=my-profile" --var "region=us-east-1" launch prod/s3-synapse-sync-kms-key.yaml
```

Add two **SecureString** parameters containing Synapse credentials to SSM Parameter Store

| Parameter Name  | Value | Type |
| ------------- | ------------- | ------------- |
| `/HTAN/SynapseSync/username`  | Synapse service account username  | SecureString |
| `/HTAN/SynapseSync/apiKey`  | Synapse service account API Key | SecureString |

```shell script
aws ssm put-parameter \
  --name /HTAN/SynapseSync/username \
  --value <synapse user name> \
  --type SecureString \
  --key-id alias/s3-synapse-sync-kms-key/kmskey
```

### Create a local build

```shell script
$ sam build
```

### Run locally

```shell script
$ sam local invoke HelloWorldFunction --event events/event.json
```

### Run unit tests
Tests are defined in the `tests` folder in this project. Use PIP to install the
[pytest](https://docs.pytest.org/en/latest/) and run unit tests.

```shell script
$ python -m pytest tests/ -v
```

## Deployment

### Deploy Lambda to S3
Deployments are sent to the
[Sage cloudformation repository](https://bootstrap-awss3cloudformationbucket-19qromfd235z9.s3.amazonaws.com/index.html)
which requires permissions to upload to Sage
`bootstrap-awss3cloudformationbucket-19qromfd235z9` and
`essentials-awss3lambdaartifactsbucket-x29ftznj6pqw` buckets.

```shell script
sam package --template-file .aws-sam/build/template.yaml \
  --s3-bucket essentials-awss3lambdaartifactsbucket-x29ftznj6pqw \
  --output-template-file .aws-sam/build/s3-synapse-sync.yaml

aws s3 cp .aws-sam/build/s3-synapse-sync.yaml s3://bootstrap-awss3cloudformationbucket-19qromfd235z9/s3-synapse-sync/master/
```

## Publish Lambda

### Private access
Publishing the lambda makes it available in your AWS account.  It will be accessible in
the [serverless application repository](https://console.aws.amazon.com/serverlessrepo).

```shell script
sam publish --template .aws-sam/build/cfn-cr-synapse-tagger.yaml
```

### Public access
Making the lambda publicly accessible makes it available in the
[global AWS serverless application repository](https://serverlessrepo.aws.amazon.com/applications)

```shell script
aws serverlessrepo put-application-policy \
  --application-id <lambda ARN> \
  --statements Principals=*,Actions=Deploy
```

## Install Lambda into AWS

### Sceptre
Create the following [sceptre](https://github.com/Sceptre/sceptre) file

Create a sceptre s3-synapse-sync.yaml file used to deploy cloudformation
template [s3-synapse-sync.yaml](template.yaml):
```yaml
template_path: "remote/s3-synapse-sync.yaml"
stack_name: "s3-synapse-sync"
stack_tags:
  Department: "CompOnc"
  Project: "HTAN"
  OwnerEmail: "joe.smith@sagebase.org"
dependencies:
  - "prod/s3-synapse-sync-kms-key.yaml"
hooks:
  before_launch:
    - !cmd "curl https://{{stack_group_config.admincentral_cf_bucket}}.s3.amazonaws.com/s3-synapse-sync/master/s3-synapse-sync.yaml --create-dirs -o templates/remote/s3-synapse-sync.yaml"
parameters:
  BucketVariables: >-
    {
      "htan-dcc-bucket-a":{"SynapseProjectId":"syn11111"},
      "htan-dcc-bucket-b":{"SynapseProjectId":"syn22222"}
    }
  KmsDecryptPolicyArn: !stack_output_external "s3-synapse-sync-kms-key::KmsDecryptPolicyArn"
  BucketNamePrefix: "htan-dcc-*"
```

Install the lambda using sceptre:
```shell script
sceptre --var "profile=my-profile" --var "region=us-east-1" launch prod/s3-synapse-sync.yaml
```

### AWS Console
Steps to deploy from AWS console.

1. Login to AWS
2. Access the
[serverless application repository](https://console.aws.amazon.com/serverlessrepo)
-> Available Applications
3. Select application to install
4. Enter Application settings
5. Click Deploy

---
## Create Buckets
**Note**: Buckets must be explicitly named. Bucket names must begin with the prefix specified in the lambda parameter `BucketNamePrefix` (e.g. htan-dcc-*) and must be globally unique across all AWS accounts.

Create a sceptre s3-synapse-sync-bucket-a.yaml file used to deploy jinjaized
cloudformation template [s3-synapse-sync-bucket-a.yaml](s3-synapse-sync-bucket.j2):
```yaml
template_path: "remote/s3-synapse-sync-bucket.j2"
stack_name: "s3-synapse-sync-bucket-a"
stack_tags:
  Department: "CompOnc"
  Project: "HTAN"
  OwnerEmail: "joe.smith@sagebase.org"
hooks:
  before_launch:
    - !cmd "curl https://{{stack_group_config.admincentral_cf_bucket}}.s3.amazonaws.com/s3-synapse-sync/master/s3-synapse-sync-bucket.j2 --create-dirs -o templates/remote/s3-synapse-sync-bucket.j2"
dependencies:
  - "prod/s3-synapse-sync.yaml"
parameters:
  BucketName: "htan-dcc-bucket-a"
  SynapseIDs:
    - "1111111"
  S3UserARNs:
    - "arn:aws:sts::213235685529:assumed-role/sandbox-developer/joe.smith@sagebase.org"
  DenyDeleteARNs:
    - "arn:aws:sts::213235685529:assumed-role/sandbox-developer/joe.smith@sagebase.org"
  S3SynapseSyncFunctionArn: !stack_output_external "s3-synapse-sync::FunctionArn"
  S3SynapseSyncFunctionRoleArn: !stack_output_external "s3-synapse-sync::FunctionRoleArn"

# Due to circular dependencies, enabling bucket notification must be done after bucket creation"
# https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-properties-s3-bucket-notificationconfig.html
sceptre_user_data:
  EnableNotificationConfiguration: "false"
```

Deploy with sceptre, Notification configuration is disabled on 1st deploy.
Deploy a 2nd time with `EnableNotificationConfiguration: "true"`

---

### To Use:
1. Place file in folder of S3 bucket
    - Grant the bucket owner full control of the object by including the flag `--acl bucket-owner-full-control`

Example `cp` and `put-object` commands:
```
aws s3 cp test.txt s3://MyBucket/MyFolder/test.txt --acl bucket-owner-full-control
```
```
aws s3api put-object --bucket MyBucket --key MyFolder/test.txt --body test.txt --acl bucket-owner-full-control
```

2. Check CloudWatch logs for the Lambda function to see if the function was triggered and completed successfully
3. Check Synapse project to see if filehandle was created

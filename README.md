Install sam cli and aws cli.
Enable long paths in windows.

https://catalog.workshops.aws/complete-aws-sam/en-US/module-1-sam-setup/10-initialize

=====================================================

can invoke lambda locally by 2 ways (these create .aws-sam folder):

1. Invoke once with
  sam local invoke --event events/event.json

2. Simulate an api gateway using docker. This is live reload.
  sam local start-api --port 8080

======================================================
Run the unit tests

cd into hello-world
  npm run test

========================================================

Manual deployments

The artifacts that are created are:

1. zip file of your code which gets uploaded to s3

2. packaged template which is a copy of the template.yaml, except the codeUri references the zip that was uploaded to s3

To build these artifacts
  sam build

This creates a new folder, .aws-sam. This contains a build folder and node_modules

In cmd, login to aws with (to login to aws console, MFA goes to authenticator app on phone)

To set up aws-cli, go to iam and create an access key
then in cmd, 
  aws configure
input the access key and secret key

Deploy the app with
  sam deploy --guided

This creates aws-sam-cli-managed-default (s3 bucket) and sam-app (lambdas, api gateways, iam roles) stack in cloudformation.

Run with --guided if running for the first time so it will save the configuration to samconfig.toml.

For AWS Region, use us-west-1.

Make sure to answer y to the question about missing authorization: HelloWorldFunction may not have authorization defined, Is this okay? [y/N]: y

  1. Your codebase gets packaged as a zip file.
  2. SAM creates an S3 bucket in your account, if it doesn't already exist.
  3. Zip file is uploaded to the S3 bucket.
  4. SAM creates the packaged template that references the location of the zip file on S3.
  5. The packaged template is also uploaded to the S3 bucket.
  6. SAM starts the deployment via CloudFormation ChangeSets.

To delete stack
  sam delete


=========================================================

CI/CD

Can generate deployment pipelines with

AWS CodePipeline, Github Actions, Jenkins

AWS CodePipeline method:
cd into sam-app
Then
  sam pipeline init --bootstrap
Then follow this
  https://catalog.workshops.aws/complete-aws-sam/en-US/module-4-cicd/module-4-cicd-codepipeline/50-sampipeinit#create-cloudformation-pipeline-template

This will create the following required resources for the 'dev' configuration aka aws-sam-cli-managed-dev-pipeline-resources:
        - Pipeline IAM user
        - Pipeline execution role
        - CloudFormation execution role
        - Artifact bucket

Do this for prod as well.

The difference from the tutorial is the Git provider, which for me it's GitHub.

What is the Git branch used for production deployments? master

This will create codepipeline.yaml, assume-role.sh, /pipeline.

Deploy the cloudformation stack of the pipeline aka sam-app-pipeline (not actual stack of the hello-world app)
  sam deploy -t codepipeline.yaml --stack-name sam-app-pipeline --capabilities=CAPABILITY_IAM

Got an error
  Connection GetRepository Connection is not available aws codepipeline

Needed to go into CodePipeline console and edit the source and manually approve the github app connection.

Action Provider should be GitHub (Version 2)

Now getting an error in the BuildAndPackage step of the pipeline.
  An error occurred (AccessDenied) when calling the AssumeRole operation: User: arn:aws:sts::597043972440:assumed-role/sam-app-pipeline-CodeBuildServiceRole-CPzfCZy69auL/AWSCodeBuild-96b222ee-2cb8-49b3-a17c-d140c0e394d9 is not authorized to perform: sts:AssumeRole on resource: arn:aws:iam::597043972440:role/aws-sam-cli-managed-dev-pipel-PipelineExecutionRole-kNooKloc8vjb

Now trying to delete sam-app-pipeline stack with
  aws cloudformation delete-stack  --stack-name sam-app-pipeline
But get this error
  An error occurred (ValidationError) when calling the DeleteStack operation: Role arn:aws:iam::597043972440:role/sam-app-pipeline-PipelineStackCloudFormationExecuti-SSL7OT1Acdlp is invalid or cannot be assumed

Going in cloudformation console, and clicking the url to that role, it cannot be found.
Might have to recreate that role (arn:aws:iam::597043972440:role/sam-app-pipeline-PipelineStackCloudFormationExecuti-SSL7OT1Acdlp), as mentioned in
  https://thewerner.medium.com/aws-cloud-formation-role-arn-aws-iam-xxx-is-invalid-or-cannot-be-assumed-14c17e1098e2

Create an iam_role_cfn.json file.
Create that role with aws-cli
  aws iam create-role --role-name sam-app-pipeline-CodePipelineExecutionRole-SjVyCbQ0m1Ws --assume-role-policy-document file://iam_role_cfn.json --description "TEMP ROLE: Allow CFN to administer 'zombie' stack"

And then attach the Admin policy ARN:
  aws iam attach-role-policy --role-name sam-app-pipeline-CodePipelineExecutionRole-SjVyCbQ0m1Ws --policy-arn arn:aws:iam::aws:policy/AdministratorAccess

Now when doing
  aws cloudformation delete-stack  --stack-name sam-app-pipeline
it doesn't give any error. When checking cloudformation, stack is still there, but there is a new role. Might have to do it with the other role that failed to delete.

Repeat above with the other role.
Create that role with aws-cli
  aws iam create-role --role-name sam-app-pipeline-PipelineStackCloudFormationExecuti-1q9vLfj4qLFY --assume-role-policy-document file://iam_role_cfn.json --description "TEMP ROLE: Allow CFN to administer 'zombie' stack"

And then attach the Admin policy ARN:
  aws iam attach-role-policy --role-name sam-app-pipeline-PipelineStackCloudFormationExecuti-1q9vLfj4qLFY --policy-arn arn:aws:iam::aws:policy/AdministratorAccess

Still fails. The two roles cannot be found again. Maybe there are race conditions.

  User: arn:aws:iam::782908857608:user/sts-role-assumer is not authorized to perform: sts:AssumeRole on resource: arn:aws:iam::597043972440:role/sam-app-pipeline-PipelineStackCloudFormationExecuti-SSL7OT1Acdlp (Service: AWSSecurityTokenService; Status Code: 403; Error Code: AccessDenied; Request ID: 3efea1a5-16d8-4d34-ad60-eeb106ab4213; Proxy: null)

Might have to recreate sts-role-assumer user and give admin policies.
Didn't work maybe because the user that was created has account number 597043972440, not 782908857608. Don't know where 782908857608 is from.

Managed to delete the sam-app-pipeline stack by recreating the roles that failed to delete, then going into cloudformation console, deleting stack again then selecting to retain the cloudformation role that failed to delete, not the other role.

If the aws-sam-cli-managed-default stack fails to delete, have to delete all items in the bucket first.

===========
CodePipeline doesn't work with GitHub. Trying GitHub Actions now.
https://catalog.workshops.aws/complete-aws-sam/en-US/module-4-cicd/module-4-cicd-gh/50-sampipeinit

  sam pipeline bootstrap --stage dev

This creates the .aws-sam folder. Also creates the aws-sam-cli-managed-dev-pipeline-resources stack in cloudformation.

Do the same for prod
  sam pipeline bootstrap --stage prod

Next
  sam pipeline init
This creates the instructions for github actions in 
  .github\workflows\pipeline.yaml

Next commit
  git add .
  git commit -m "Adding SAM CI/CD Pipeline definition"
  git push

Getting an error in the GitHub Actions workflow, build-and-package step, Upload artifacts to testing artifact buckets, 
  Cannot use both --resolve-s3 and --s3-bucket parameters. Please use only one.
Try to edit samconfig.toml [default.package.parameters] resolve_s3 to false? 
Not sure if the resolve_s3 in [default.deploy.parameters] should also be false.
I guess so because it failed in the deploy-testing > Deploy to testing account step
  Cannot use both --resolve-s3 and --s3-bucket parameters in non-guided deployments. Please use only one or use the --guided option for a guided deployment.

Now errored in deploy-testing > Deploy to testing account step
  Aborted!
  Deploy this changeset? [y/N]: 
  Error: Process completed with exit code 1.

Might have to change confirm_changeset = false from true

It works now. Pushing to master deploys to dev(stage) and prod environments.
Should probably change the master to develop so it deploys to dev(stage) when merging to develop branch .github\workflows\pipeline.yaml 
  deploy-testing:
    if: github.ref == 'refs/heads/develop'

OR make a separate pipeline.yaml for dev and for prod

  on:
    push:
      branches:
        - 'develop'
        - 'feature**'
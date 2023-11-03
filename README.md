Install sam cli and aws cli.
Enable long paths in windows.

https://catalog.workshops.aws/complete-aws-sam/en-US/module-1-sam-setup/10-initialize

=====================================================

can invoke lambda locally by 2 ways:

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

This will create the following required resources for the 'dev' configuration:
        - Pipeline IAM user
        - Pipeline execution role
        - CloudFormation execution role
        - Artifact bucket

Do this for prod as well.

The difference from the tutorial is the Git provider, which for me it's GitHub.

What is the Git branch used for production deployments? master

This will create codepipeline.yaml, assume-role.sh, /pipeline.
# Running nextflow jobs on AWS Batch

https://www.nextflow.io/docs/latest/awscloud.html


## Setting up a batch queue

Create an execution environment https://docs.aws.amazon.com/cli/latest/reference/batch/create-compute-environment.html

Get (or create) subnets:

    aws ec2 describe-subnets --query 'Subnets[].SubnetId'

Get the default security group (or alternative create a new one):

    aws ec2 describe-security-groups --group-names default

Get the AWS Batch role ARN (this can be created through the AWS console but you can also create it manually https://docs.aws.amazon.com/batch/latest/userguide/service_IAM_role.html)

    aws iam get-role --role-name AWSServiceRoleForBatch

Edit [`batch-compute-environment.json`](./batch-compute-environment.json), replace `"SUBNET_IDS"`, `"SECURITY_GROUP_IDS"` and `"AWS_BATCH_SERVICE_ROLE_ARN"` (https://docs.aws.amazon.com/batch/latest/userguide/fargate.html).

    aws batch create-compute-environment --cli-input-json file://batch-compute-environment.json

Create the job queue

   aws batch create-job-queue --job-queue-name TEST-nextflow-batch-queue --state ENABLED --priority 1 --compute-environment-order order=1,computeEnvironment=TEST-nextflow-batch-compute


## Running

    nextflow run tutorial.nf -bucket-dir s3://<BUCKET>/some/path

Note if you are using temporary AWS session credentials then [setting them with environment variables (`AWS_ACCESS_KEY_ID` `AWS_SECRET_ACCESS_KEY` `AWS_SESSION_TOKEN`) does not work](https://github.com/nextflow-io/nextflow/issues/1724). Instead you should add the temporary credentials to your `~/.aws/credentials` file and set `AWS_PROFILE=<profile-name>`.


## Clean up

Delete the compute environment

    aws batch update-job-queue --job-queue TEST-nextflow-batch-queue --state DISABLED
    aws delete-job-queue --job-queue TEST-nextflow-batch-queue
    aws batch update-compute-environment --compute-environment TEST-nextflow-batch-compute --state DISABLED
    aws batch delete-compute-environment --compute-environment TEST-nextflow-batch-compute


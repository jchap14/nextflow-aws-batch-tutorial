# Running nextflow jobs on AWS Batch

Notes on getting a Nextflow pipeline to run on AWS Batch
https://www.nextflow.io/docs/latest/awscloud.html

Note Nextflow [does not support Fargate](https://groups.google.com/g/nextflow/c/JFneg8d3x2w?pli=1), so you must use `EC2` or `EC2_SPOT` types.


## Setting up a batch queue

Create an execution environment https://docs.aws.amazon.com/cli/latest/reference/batch/create-compute-environment.html

Get (or create) subnets:

    aws ec2 describe-subnets --query 'Subnets[].SubnetId'

Get the default security group (or alternative create a new one):

    aws ec2 describe-security-groups --group-names default

Get the AWS Batch role ARN (this can be automatically created through the AWS console by creating and deleting a batch compute environment but you can also [create it manually](https://docs.aws.amazon.com/batch/latest/userguide/service_IAM_role.html)).

    aws iam get-role --role-name AWSServiceRoleForBatch

Check you have the AWS ECS instance and spot fleet roles.
These can be automatically created through the AWS console by creating and deleting an ECS spot cluster but you can also create it manually: [ECS instance role](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/instance_IAM_role.html), [ECS spot fleet role](https://docs.aws.amazon.com/batch/latest/userguide/spot_fleet_IAM_role.html).

    aws iam get-role --role-name ecsInstanceRole
    aws iam get-role --role-name ecsSpotFleetRole

However the `ecsInstanceRole` does not contain the S3 permissions required by Nextflow, so you either need to augment that role, or preferably create a new role and instance profile `nextflowEcsInstanceRole`:

    aws iam create-role --role-name nextflowEcsInstanceRole --assume-role-policy-document file://nextflowEcsInstanceRole-assume-role-policy.json
    aws iam attach-role-policy --role-name nextflowEcsInstanceRole --policy-arn arn:aws:iam::aws:policy/service-role/AmazonEC2ContainerServiceforEC2Role
    aws iam attach-role-policy --role-name nextflowEcsInstanceRole --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess

    aws iam create-instance-profile --instance-profile-name nextflowEcsInstanceRole
    aws iam add-role-to-instance-profile --instance-profile-name nextflowEcsInstanceRole --role-name nextflowEcsInstanceRole

The above role has full S3 access, in production you may want to limit access to just one bucket.

Edit [`batch-compute-environment.json`](./batch-compute-environment.json), replace:
  - `SUBNET_IDS`
  - `SECURITY_GROUP_IDS`
  - `AWS_BATCH_SERVICE_ROLE_ARN`

Now create the compute environment:

    aws batch create-compute-environment --cli-input-json file://batch-compute-environment-spot.json

Create the job queue

   aws batch create-job-queue --job-queue-name TEST-nextflow-batch-queue --state ENABLED --priority 1 --compute-environment-order order=1,computeEnvironment=TEST-nextflow-batch-compute-m4


## Storage bucket

Nextflow with AWS Batch requires an S3 location to store its outputs.
If you don't already have a location create a new bucket:

    aws s3 mb s3://BUCKET_NAME


## Running

    nextflow run tutorial.nf -bucket-dir s3://BUCKET_NAME/some/path

Note if you are using temporary AWS session credentials then [setting them with environment variables (`AWS_ACCESS_KEY_ID` `AWS_SECRET_ACCESS_KEY` `AWS_SESSION_TOKEN`) does not work](https://github.com/nextflow-io/nextflow/issues/1724). Instead you should add the temporary credentials to your `~/.aws/credentials` file and set `AWS_PROFILE=<profile-name>`.


## Fetching results

List all files in the S3 bucket recursively:

    aws s3 ls --recursive s3://BUCKET_NAME/some/path

Copy all files

    aws cp --recursive s3://BUCKET_NAME/some/path dest

## Clean up

Delete the compute environment

    aws batch update-job-queue --job-queue TEST-nextflow-batch-queue --state DISABLED
    aws delete-job-queue --job-queue TEST-nextflow-batch-queue
    aws batch update-compute-environment --compute-environment TEST-nextflow-batch-compute --state DISABLED
    aws batch delete-compute-environment --compute-environment TEST-nextflow-batch-compute


## Additional options
- Specify a launch template in the compute environment to custommise an AMI at launch time without rebuilding
https://docs.aws.amazon.com/batch/latest/userguide/launch-templates.html

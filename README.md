# S3 Intelligent-Tiering Bucket Policy Enforcement

## Description
[Amazon Simple Storage Service (S3) Intelligent-Tiering](https://aws.amazon.com/s3/storage-classes/intelligent-tiering/) is designed to optimize storage costs by automatically moving data between multiple access tiers.  Less frequently accessed objects will be moved to colder tier which has smaller per Gigabyte storage cost.

S3 Intelligent-Tiering deserves to be the default storage class when using S3.  Because the default three access tiers, 1/ **Frequent Access Tier**, 2/ **Infrequent Access Tier**, and 3/ **Archive Instant Access Tier**, provide low latency access to the objects, and there is no fee for moving objects between tiers and retrieval of the data.  Just to be sure, there is a monitoring fee for objects bigger than 128 Kilobytes ($0.0025 per 1,000 objects).

Enforcing S3 Intelligent-Tiering for storage class selection requires [setting a bucket policy](https://docs.aws.amazon.com/AmazonS3/latest/userguide/amazon-s3-policy-keys.html#example-storage-class-condition-key).  This cannot be achieved with [AWS Identity and Access Management (IAM)](https://aws.amazon.com/iam/) or [service control policy (SCP)](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_policies_scps.html).  This is a sample for applying the following bucket policy to all S3 buckets in the account at once.

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": <statement-id>,
            "Effect": "Deny",
            "Principal": {
                "AWS": "*"
            },
            "Action": "s3:PutObject",
            "Resource": "arn:aws:s3:::<bucket-name>/*",
            "Condition": {
                "StringNotEquals": {
                    "s3:x-amz-storage-class": ["INTELLIGENT_TIERING"]
                }
            }
        }
    ]
}
```

Two [AWS Lambda](https://aws.amazon.com/lambda/) functions are created by the [AWS CloudFormation](https://aws.amazon.com/cloudformation/) template.

**FunctionToAddS3BucketPolicy** (CloudFormation resource name) can apply the above bucket policy in bulk.  By executing the other **FunctionToDeleteS3BucketPolicy** (CloudFormation resource name), you can delete all the bucket policies applied by **FunctionToAddS3BucketPolicy**.

## Installation
This sample can be deployed via CloudFormation template.  The only parameter that must be specified is the S3 bucket name which stores execution results of Lambda functions.

## Usage
The two Lambda functions **FunctionToAddS3BucketPolicy** and **FunctionToDeleteS3BucketPolicy** can both be executed by themselves.  These do not refer to `event` and `context` passed to the Lambda handler, so it is possible to add and remove bucket policies in the **Test** execution.  The execution results are recorded in the specified S3 bucket in the following format as csv.

|account_id|bucket_name|status|
|:-:|:-:|:-:|
|123456789012|bucket_name-01|Added a new S3 Bucket Policy|
|123456789012|bucket_name-02|No changes made to S3 Bucket Policy|

The CloudFormation template also creates an [Amazon EventBridge](https://aws.amazon.com/eventbridge/) Rule for periodic execution.  This EventBridge Rule is created in a **Disabled** state.  With this **Enabled**, the Lambda function **FunctionToAddS3BucketPolicy** will run every Sunday to scan all S3 buckets in your AWS account and attempt to apply the bucket policy.

Just to be safe, this sample does not make any changes to S3 buckets that already have a bucket policy set.

## Support
This sample was tested that the expected results were obtained in an actual AWS account.  It is implemented so that no changes are made if the bucket policy is already applied, but we recommend that you try it once in a test environment when using it.

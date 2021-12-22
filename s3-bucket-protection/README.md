![CrowdStrike](https://raw.github.com/CrowdStrike/Cloud-AWS/main/docs/img/cs-logo.png)

[![Twitter URL](https://img.shields.io/twitter/url?label=Follow%20%40CrowdStrike&style=social&url=https%3A%2F%2Ftwitter.com%2FCrowdStrike)](https://twitter.com/CrowdStrike)

# CrowdStrike Falcon S3 Bucket Protection

+ [Overview](#overview)
+ [Integration components](#integration-components)
+ [Demonstration](#running-the-demonstration)

## Overview
This solution integrates CrowdStrike Falcon Quick Scan with AWS S3, AWS Security Hub and AWS Systems Manager (Parameter Store), allowing for files to be scanned and threats remediated as objects are added to the bucket.

### Process diagram
![Process Diagram](../docs/img/s3-bucket-protection-process-diagram.png)

1. Files are uploaded to the bucket
2. Bucket triggers the lambda function
3. Lambda function reads in Falcon API credentials from Systems Manager Parameter Store
4. Lambda function uploads file to Falcon X Sandbox for analysis
5. Lambda function retrieves the scan results
6. Malicious files are immediately removed from the bucket
7. A finding is generated in Security Hub for all malicious uploads

> All lambda activity is also logged to Amazon CloudWatch

---

## Integration components
This solution leverages an S3 bucket trigger to call AWS Lambda for processing. 
The necessary resources and permissions to implement this functionality are described below.

### AWS S3 
- Bucket
- Bucket notification `s3:ObjectCreated:*` -> Lambda trigger

### AWS Lambda function (Python 3)
- `crowdstrike-falconpy` layer
- Policy statement
    - Statement ID: `AllowExecutionFromS3Bucket`
    - Principal: `s3.amazonaws.com`
    - Effect: `Allow`
    - Action: `lambda:InvokeFunction`
    - Conditions
      ```json
      {
        "ArnLike": {
            "AWS:SourceArn": "arn:aws:s3:::{LAMBDA_FUNCTION_NAME}"
        }
      }
      ```
- Execution Role (Detailed in IAM section)
- Environment variables
    - `base_url`: CrowdStrike API base URL (Only required for GovCloud)
    - `CLIENT_ID_PARAM`: Name of the Parameter store parameter containing the CrowdStrike API Key.
    - `CLIENT_SECRET_PARAM`: Name of the Parameter store parameter containing the CrowdStrike API Secret.

### AWS IAM
- Lambda execution role
    - Security Hub policy
      ```json
      {
        "Statement": [
            {
                "Action": "securityhub:GetFindings",
                "Effect": "Allow",
                "Resource": "arn:aws:securityhub:{REGION}:{ACCOUNT_ID}:hub/default",
                "Sid": ""
            },
            {
                "Action": "securityhub:BatchImportFindings",
                "Effect": "Allow",
                "Resource": "arn:aws:securityhub:{REGION}:517716713836:product/crowdstrike/*",
                "Sid": ""
            }
        ],
        "Version": "2012-10-17"
      }
      ```
    - S3 policy
      ```json
      {
        "Statement": [
            {
                "Action": [
                    "s3:GetObjectVersion",
                    "s3:GetObject",
                    "s3:DeleteObjectVersion",
                    "s3:DeleteObject"
                ],
                "Effect": "Allow",
                "Resource": "arn:aws:s3:::{BUCKET_NAME}/*",
                "Sid": ""
            }
        ],
        "Version": "2012-10-17"
      }
      ```
    - SSM policy
      ```json
      {
        "Statement": [
            {
                "Action": [
                    "ssm:GetParametersByPath",
                    "ssm:GetParameters",
                    "ssm:GetParameterHistory",
                    "ssm:GetParameter"
                ],
                "Effect": "Allow",
                "Resource": "arn:aws:ssm:{REGION}:{ACCOUNT_ID}:parameter/*",
                "Sid": ""
            }
        ],
        "Version": "2012-10-17"
      }
      ```
    - Execution policy
      ```json
      {
        "Statement": [
            {
                "Action": "logs:CreateLogGroup",
                "Effect": "Allow",
                "Resource": "arn:aws:logs:{REGION}:{ACCOUNT_ID}:*",
                "Sid": ""
            },
            {
                "Action": [
                    "logs:PutLogEvents",
                    "logs:CreateLogStream"
                ],
                "Effect": "Allow",
                "Resource": "arn:aws:logs:{REGION}:{ACCOUNT_ID}:log-group:/aws/lambda/{LAMBDA_FUNCTION_NAME}:*",
                "Sid": ""
            }
        ],
        "Version": "2012-10-17"
      }
      ```

### AWS Systems Manager
- Parameter Store parameters
    - CrowdStrike API Key (SecureString)
    - CrowdStrike API Secret (SecureString)

---

## Running the demonstration
The demonstration leverages Terraform to configure the environment.

### Requirements

+ AWS account access with appropriate CLI keys and permissions already configured.
+ The pre-existing PEM key for SSH access to the demonstration instance. This key must exist within the region your demonstration stands up in. (Default: `us-east-2`)
    - You will be asked for the name of this key when the `demo.sh` script executes.
+ CrowdStrike Falcon API credentials with the following scopes:
    - Quick Scan - `READ`, `WRITE`
    - Sample Uploads - `READ`,`WRITE`
    - You will be asked to provide these credentials when the `demo.sh` script executes.
+ Terraform installed on the machine you are testing from.
+ The current external IP address of the machine you are testing from.

### Additional components
In order to demonstrate functionality, this demonstration adds the additional AWS components on top of the components listed above.

#### AWS VPC
+ VPC
    - Single public subnet
    - Internet gateway
    - Public route table
    - Security group allowing SSH access for defined Trusted IP

#### AWS EC2
+ EC2 Instance, `t2.small`
    - IAM Role with two policies
        - Security Hub access
        - S3 access to the bucket (explicitly)
    - Custom helper scripts to demonstrate functionality
    - Sample files for bucket testing
    - Attached to the VPC subnet
    - Attached to the SSH inbound security group


### Standing up the demonstration
From this folder execute the following command:

```shell
./demo.sh up
```

You will be asked to provide:
+ Your CrowdStrike API credentials.
    - These values will not display when entered.
+ The name of the PEM key to use for SSH access.
+ Your external IP address.

```shell
√ s3-bucket-protection % ./demo.sh up

This script should be executed from the s3-bucket-protection root directory.

CrowdStrike API Client ID:
CrowdStrike API Client SECRET:
The following values are not required for the integration, only the demo.
EC2 Instance Key Name: instance-key-name
Trusted IP address: a.b.c.d
```

If this is the first time you're executing the demonstration, Terraform will initialize the working folder after you submit these values. After this process completes, Terraform will begin to stand-up the environment.

The demonstration instance is assigned an external IP address, and a security group that allows inbound SSH access from the trusted IP address you provide. The username and external IP address for this instance will be provided when environment creation is complete.

The name of the bucket used for the demonstration is randomly generated and will be output when environment creation is complete.

To stand up the entire demonstration takes approximately two minutes, after which you will be presented with the message:

```shell
  __                        _
 /\_\/                   o | |             |
|    | _  _  _    _  _     | |          ,  |
|    |/ |/ |/ |  / |/ |  | |/ \_|   |  / \_|
 \__/   |  |  |_/  |  |_/|_/\_/  \_/|_/ \/ o
```

### Using the demonstration
Once the environment has completed standing up, you should be able to connect via SSH immediately.

```shell
ssh -i KEY_LOCATION ec2-user@IP_ADDRESS
```

Upon successful login, the following help menu should be displayed:

```shell
 _______                        __ _______ __        __ __
|   _   .----.-----.--.--.--.--|  |   _   |  |_.----|__|  |--.-----.
|.  1___|   _|  _  |  |  |  |  _  |   1___|   _|   _|  |    <|  -__|
|.  |___|__| |_____|________|_____|____   |____|__| |__|__|__|_____|
|:  1   |                         |:  1   |
|::.. . |                         |::.. . |
`-------'                         `-------'

Welcome to the CrowdStrike Falcon S3 Bucket Protection demo environment!

The name of your test bucket is efdcecfa-s3-protected-bucket and is
available in the environment variable BUCKET.

There are test files in the testfiles folder.
Use these to test the lambda trigger on bucket uploads.
NOTICE: Files labeled `malicious` are DANGEROUS!

Use the command `upload` to upload all of the test files to your demo bucket.

You can view the contents of your bucket with the command `list-bucket`.

Use the command `get-findings` to view all findings for your demo bucket.
```

#### Helper Commands
There are several helper scripts implemented within the environment to demonstrate command line usage.

##### get-findings
Displays any negative findings for the demonstration bucket.

##### list-bucket
Lists the contents of the demonstration bucket.

##### upload
Uploads the entire contents of the testfiles folder to the demonstration bucket.

This folder is located in `~/testfiles` and contains multiple samples named according to the sample's type.
+ 2 safe sample files
+ 3 malware sample files
+ 2 unscannable sample files

#### Console example
The following screenshots demonstrate the same functionality using the AWS console.

##### Reviewing Security Hub findings
![Findings](../docs/img/s3-bucket-protection-finding-list.png)

##### Reviewing finding details
![Finding details](../docs/img/s3-bucket-protection-finding-display.png)

### Tearing down the demonstration
From the same folder where you stood up the environment, execute the command:

```shell
./demo down
```

You will receive no further prompts.

Once the environment has been destroyed, you will be provided the message:

```shell
 ___                              __,
(|  \  _  , _|_  ,_        o     /  |           __|_ |
 |   ||/ / \_|  /  | |  |  |    |   |  /|/|/|  |/ |  |
(\__/ |_/ \/ |_/   |/ \/|_/|/    \_/\_/ | | |_/|_/|_/o
```
---
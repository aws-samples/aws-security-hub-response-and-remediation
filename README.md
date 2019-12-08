## Response and Remediation with AWS Security Hub

This repo contains the CloudFormation template to deploy Security Hub custom actions, CloudWatch Event Rules and Lambda functions as detailed in the AWS Security Blog Post: Response and remediation with AWS Security Hub. There is an FAQ below that details some other information and upcoming plans for this project to extend the Blog.

***QUICK NOTE:*** The end-to-end combination of a Security Hub custom action, a CloudWatch Event rule, a Lambda function, plus any supporting services needed to perform a specific action is referred to as a “playbook.”

### Getting Started

Download the AWS CloudFormation template (`SecurityHub_CISPlaybooks_CloudFormation.yaml`) and deploy it from the console

### Solutions Architecture
![Architecture](https://github.com/aws-samples/aws-security-hub-remediation-code/blob/master/Architecture.jpg)
1.	Integrated services send their findings to Security Hub.
2.	From the Security Hub console, you’ll choose a custom action for a finding. Each custom action is then emitted as a CloudWatch Event.
3.	The CloudWatch Event rule triggers a Lambda function. This function is mapped to a custom action, based on the custom action’s ARN.
4.	Dependent on the particular rule, the Lambda function  invoked will perform a remediation action on your behalf

### What remediation playbooks are covered and how?
All playbooks are made up of CloudWatch Events & Lambda functions, details are as follows:

#### CIS AWS Benchmark Playbooks (10 Playbooks):
-	**1.3 – “Ensure credentials unused for 90 days or greater are disabled”**
-	**1.4 – “Ensure access keys are rotated every 90 days or less”**
    - A Lambda function will loop through the User's access key as identified in the finding, any keys that are over the age of 90 days will be deactivated and if successful a Note will be added to the Security Hub finding.
-	**1.5 – “Ensure IAM password policy requires at least one uppercase letter”**
-	**1.6 – “Ensure IAM password policy requires at least one lowercase letter”**
-	**1.7 – “Ensure IAM password policy requires at least one symbol”**
-	**1.8 – “Ensure IAM password policy requires at least one number”**
-	**1.9 – “Ensure IAM password policy requires a minimum length of 14 or greater”**
-	**1.10 – “Ensure IAM password policy prevents password reuse”**
-	**1.11 – “Ensure IAM password policy expires passwords within 90 days or less”**
    - A Lambda function will call the IAM UpdateAccountPasswordPolicy API with CIS-compliant parameters for the password policy, due to the fact no inputs are expected, a note will not be added to any finding.
-	**2.2 – “Ensure CloudTrail log file validation is enabled”**
    - A Lambda function will parse out the CloudTrail information from the finding and call the UpdateTrail API to turn log file validation back on, if successful a Note will be added to the Security Hub finding. 
-	**2.3 – “Ensure the S3 bucket CloudTrail logs to is not publicly accessible”**
    - A Lambda function will parse out the S3 Bucket information from the finding and calls the Systems Manager StartAutomationExecution API to run the Automation document `AWS-DisableS3BucketPublicReadWrite` to remove public Read & Write access from the bucket. You will need to follow-up in the Automation console, if the automation is executed successfully a Note will be added to the Security Hub finding.
-	**2.4 – “Ensure CloudTrail trails are integrated with Amazon CloudWatch Logs”**
    - To automatically ensure your CloudTrail logs are sent to CloudWatch, the Lambda function for this playbook will create a brand new CloudWatch Logs group that has the name of the non-compliant CloudTrail trail in it for easy identification. The Lambda function will programmatically update your non-compliant CloudTrail trail to send its logs to the newly created log group.  To accomplish this, CloudTrail needs an IAM role and permissions to be allowed to publish logs to CloudWatch. To avoid creating multiple new IAM roles and policies via Lambda, you’ll populate the ARN of this IAM role in the Lambda environmental variables for this playbook. If successful a Note will be added to the Security Hub finding.
-	**2.6 – “Ensure S3 bucket access logging is enabled on the CloudTrail S3 bucket”**
    - To ensure the S3 bucket that contains your CloudTrail logs has access logging enabled, the Lambda function for this playbook invokes the Systems Manager document `AWS-ConfigureS3BucketLogging` this document will enable access logging for that bucket. To avoid statically populating your S3 access logging bucket in the Lambda function’s code, you’ll pass that value in via an environmental variable. You will need to follow-up in the Automation console, if the automation is executed successfully a Note will be added to the Security Hub finding.
-	**2.7 – “Ensure CloudTrail logs are encrypted at rest using AWS KMS CMKs”**
    - The Code and Instructions are provided in the Blog post 
-	**2.8 – “Ensure rotation for customer created CMKs is enabled”**
    - A Lambda function will parse out the KMS CMK infromation from the finding and call the KMS EnableKeyRotation API to enable rotation and if successful a Note will be added to the Security Hub finding.
-	**2.9 – “Ensure VPC flow logging is enabled in all VPCs”**
    - To enable VPC flow logging for rejected packets, the Lambda function for this playbook will create a new CloudWatch Logs group. For easy identification, the name of the group will include the non-compliant VPC name. The Lambda function will programmatically update your VPC to enable flow logs to be sent to the newly created log group. Similar to CloudTrail logging, VPC flow log need an IAM role and permissions to be allowed to publish logs to CloudWatch. To avoid creating multiple new IAM roles and policies via Lambda, you’ll populate the ARN of this IAM role in the Lambda environmental variables for this playbook, and if successful a Note will be added to the Security Hub finding.
-	**4.1 – “Ensure no security groups allow ingress from 0.0.0.0/0 to port 22”**
-	**4.2 – “Ensure no security groups allow ingress from 0.0.0.0/0 to port 3389”**
    - A Lambda function will parse out the Security Group information from the finding and calls the Systems Manager StartAutomationExecution API to run the Automation document `AWS-DisablePublicAccessForSecurityGroup` to remove 0.0.0.0/0 rules from the Security Group. You will need to follow-up in the Automation console, if the automation is executed successfully a Note will be added to the Security Hub finding.


#### Custom Playbooks (2 Playbooks):
- **Send Findings to JIRA**
    - A Lambda function calls the Systems Manager StartAutomationExecution API to run the Automation document `AWS-CreateJiraIssue` JIRA information is provided via Lambda env vars
- **Apply Patch Baseline**
    - A Lambda function calls the Systems Manager SendCommand API to invoke the `AWS-UpdateSSMAgent` and `AWS-RunPatchBaseline` Documents on the instance. Because you can realistically call this playbook from any finding that specifies AwsEc2Instance as its Resource.Type (GuardDuty findings, Inspector findings, 3rd party product findings) a Note will not be added.

### How do I perform response and remediation automatically?
You can modify your CloudWatch Event to use the `Security Hub Findings - Imported` **detail-type** and specify the Title of the specific CIS Finding as shown below. When findings that match this pattern are encountered, they will invoke your Lambda function without having to specify a Custom Action in Security Hub. You can additional choose to filter down on other elements of the AWS Security Finding Format such as Product.Severity, Finding.Type or other elements to design automated remediation actions.

```
 {
   "source": [
     "aws.securityhub"
   ],
   "detail-type": [
     "Security Hub Findings - Imported"
   ],
   "detail": {
     "findings": {
      "Title": [
         "2.9 Ensure VPC flow logging is enabled in all VPCs"
      ]
     }
   }
 }
```
### Will other Playbooks be developed?
Yes, they will be uploaded here, most likely in different CloudFormation templates if it makes sense.

### How can I use these playbooks from my Security Hub Master Account?
Functionality for this is under development and will be offered as a separate CloudFormation template for those who want to run *only* from their Master account at a later date.

To get started on developing it yourself you can add an IAM Policy to your Lambda function's execution role that allows a Role in the Master Account to assume those Lambda functions. For more information see this Premium Support Knowledge Center post on cross-account Lambda policies: https://aws.amazon.com/premiumsupport/knowledge-center/lambda-function-assume-iam-role/

## License

This library is licensed under the MIT-0 License. See the LICENSE file.
# Lambda_Serverless_Assignment

Question 2:- 

Steps to Complete the Assignment
1. Setting Up the Environment
Prerequisites
•	AWS CLI installed and configured.
•	An S3 bucket created (e.g., s3-bucket-cleanup).
•	A basic understanding of Python, AWS Lambda, and Boto3.
Install and Configure AWS CLI
1.	Install AWS CLI (if not already installed):
sudo apt update
sudo apt install awscli -y
2.	Configure AWS CLI:
aws configure
Provide your:
o	Access Key
o	Secret Access Key
o	Default region (e.g., us-east-1)
o	Output format (e.g., json)
________________________________________
2. Upload Files to S3 with Metadata
Upload a File with creationDate Metadata
1.	Navigate to the directory where the file is stored:
cd /mnt/c/Users/Vika2/Downloads
2.	Upload the file to the S3 bucket with the creationDate metadata:
bash
Copy code
aws s3 cp index3.html s3://s3-bucket-cleanup --metadata creationdate="2021-03-16T10:16:00Z"
Verify Metadata
Check if the metadata was successfully added:
bash
Copy code
aws s3api head-object --bucket s3-bucket-cleanup --key index3.html
Expected Output:
json
Copy code
{
    "Metadata": {
        "creationdate": "2021-03-16T10:16:00Z"
    }
}
________________________________________
3. Deploy the Lambda Function
Create the Lambda Function
1.	Navigate to the AWS Lambda Console.
2.	Create a new Lambda function with:
o	Runtime: Python 3.x
o	Execution Role: Assign an IAM role with AmazonS3FullAccess.
3.	Copy and paste the following Lambda function code:
Final Lambda Function Code
python
Copy code
import boto3
from datetime import datetime, timedelta, timezone

def lambda_handler(event, context):
    bucket_name = "s3-bucket-cleanup"  # Replace with your bucket name
    days_threshold = 30  # Threshold in days
    deleted_files = []

    # Initialize S3 client
    s3 = boto3.client('s3')

    # Calculate the cutoff date
    cutoff_date = datetime.now(timezone.utc) - timedelta(days=days_threshold)

    # List objects in the bucket
    response = s3.list_objects_v2(Bucket=bucket_name)
    if 'Contents' in response:
        for obj in response['Contents']:
            file_name = obj['Key']

            # Get object metadata
            head_response = s3.head_object(Bucket=bucket_name, Key=file_name)
            metadata = head_response.get('Metadata', {})
            print(f"Metadata for {file_name}: {metadata}")

            # Check for 'creationDate' in metadata
            if 'creationdate' in metadata:
                creation_date = datetime.strptime(metadata['creationdate'], '%Y-%m-%dT%H:%M:%SZ').replace(tzinfo=timezone.utc)
                print(f"Creation date for {file_name}: {creation_date}")

                # Compare creationDate with the cutoff date
                if creation_date < cutoff_date:
                    s3.delete_object(Bucket=bucket_name, Key=file_name)
                    deleted_files.append(file_name)
            else:
                last_modified = obj['LastModified']
                if last_modified < cutoff_date:
                    s3.delete_object(Bucket=bucket_name, Key=file_name)
                    deleted_files.append(file_name)

    print(f"Deleted files: {deleted_files}")
    return {
        'statusCode': 200,
        'body': f"Deleted {len(deleted_files)} files: {deleted_files}"
    }
Deploy the Lambda Function
•	Click Deploy to save the code.
________________________________________
4. Test the Lambda Function
1.	Manually invoke the Lambda function:
o	Use the Test tab in the AWS Lambda Console.
o	Create a test event and click Test.
2.	Verify results:
o	Check CloudWatch Logs to ensure files were processed correctly.
o	Confirm that files older than 30 days have been deleted in the S3 bucket.
________________________________________
5. Troubleshooting
Error: "can't compare offset-naive and offset-aware datetimes"
•	This happens because creation_date was not timezone-aware.
•	Solution: Use .replace(tzinfo=timezone.utc) when parsing creationDate:
python
Copy code
creation_date = datetime.strptime(metadata['creationdate'], '%Y-%m-%dT%H:%M:%SZ').replace(tzinfo=timezone.utc)
File Not Deleted
•	Check if the file's creationDate is older than the cutoff date (30 days ago).
•	Verify metadata using:
bash
Copy code
aws s3api head-object --bucket s3-bucket-cleanup --key index3.html
File Missing Metadata
•	Ensure the creationDate metadata is added during upload:
bash
Copy code
aws s3 cp index3.html s3://s3-bucket-cleanup --metadata creationdate="2021-03-16T10:16:00Z"
Permission Issues
•	Ensure the Lambda function role has the required permissions:
json
Copy code
{
    "Effect": "Allow",
    "Action": [
        "s3:ListBucket",
        "s3:GetObject",
        "s3:DeleteObject"
    ],
    "Resource": [
        "arn:aws:s3:::s3-bucket-cleanup",
        "arn:aws:s3:::s3-bucket-cleanup/*"
    ]
}
________________________________________
6. Conclusion
This setup successfully automates the deletion of files older than 30 days in an S3 bucket using AWS Lambda and Boto3. The function uses creationDate metadata as the primary filter and falls back to LastModified if creationDate is unavailable.
________________________________________






Question 1: -


1.. EC2 Setup:
   - Navigate to the EC2 dashboard and create two new t2.micro instances (or any other available free-tier type).
   - Tag the first instance with a key `Action` and value `Auto-Stop`.
   - Tag the second instance with a key `Action` and value `Auto-Start`.
2. Lambda IAM Role:
   - In the IAM dashboard, create a new role for Lambda.
   - Attach the `AmazonEC2FullAccess` policy to this role. (Note: In a real-world scenario, you would want to limit permissions for better security.)
3. Lambda Function:
   - Navigate to the Lambda dashboard and create a new function.
   - Choose Python 3.x as the runtime.
   - Assign the IAM role created in the previous step.
   - Write the Boto3 Python script to:
     1. Initialize a boto3 EC2 client.
     2. Describe instances with `Auto-Stop` and `Auto-Start` tags.
     3. Stop the `Auto-Stop` instances and start the `Auto-Start` instances.
     4. Print instance IDs that were affected for logging purposes.
4. Manual Invocation:
   - After saving your function, manually trigger it.
   - Go to the EC2 dashboard and confirm that the instances' states have changed according to their tags.

Here’s the complete code and step-by-step guide to complete the assignment:
________________________________________
Step 1: Set Up EC2 Instances
1.	Go to the AWS Management Console > EC2 Dashboard.
2.	Launch two new t2.micro instances (or any free-tier eligible type).
3.	Add tags to the instances:
Add tag- don’t delete 
o	For the first instance:
	Key: Action
	Value: Auto-Stop
o	For the second instance:
	Key: Action
	Value: Auto-Start
________________________________________
Step 2: Create an IAM Role for Lambda
1.	Navigate to the IAM Dashboard > Roles > Create Role.
2.	Select AWS Service > Lambda > Next.
3.	Attach the AmazonEC2FullAccess policy.
Note: Use a more restrictive policy in real-world applications.
4.	Complete the role creation and note the Role ARN.
________________________________________
Step 3: Create the AWS Lambda Function
1.	Go to the Lambda Dashboard > Create Function.
2.	Choose Author from scratch.
o	Name: EC2InstanceManager
o	Runtime: Python 3.x
o	Permissions: Attach the IAM role created in Step 2.
3.	Write the following Python script:
________________________________________
Lambda Function Code
python
Copy code
import boto3
import logging

# Set up logging
logger = logging.getLogger()
logger.setLevel(logging.INFO)

def lambda_handler(event, context):
    ec2 = boto3.client('ec2')
    
    # Get all EC2 instances
    response = ec2.describe_instances(Filters=[
        {'Name': 'tag:Action', 'Values': ['Auto-Stop', 'Auto-Start']}
    ])
    
    # Initialize lists for instances to start and stop
    auto_stop_instances = []
    auto_start_instances = []
    
    # Process the response
    for reservation in response['Reservations']:
        for instance in reservation['Instances']:
            for tag in instance.get('Tags', []):
                if tag['Key'] == 'Action' and tag['Value'] == 'Auto-Stop':
                    auto_stop_instances.append(instance['InstanceId'])
                elif tag['Key'] == 'Action' and tag['Value'] == 'Auto-Start':
                    auto_start_instances.append(instance['InstanceId'])
    
    # Stop instances with 'Auto-Stop' tag
    if auto_stop_instances:
        logger.info(f"Stopping instances: {auto_stop_instances}")
        ec2.stop_instances(InstanceIds=auto_stop_instances)
    else:
        logger.info("No instances found with 'Auto-Stop' tag.")
    
    # Start instances with 'Auto-Start' tag
    if auto_start_instances:
        logger.info(f"Starting instances: {auto_start_instances}")
        ec2.start_instances(InstanceIds=auto_start_instances)
    else:
        logger.info("No instances found with 'Auto-Start' tag.")
    
    return {
        "statusCode": 200,
        "body": "Instance management completed successfully!"
    }
________________________________________
Step 4: Test the Lambda Function
1.	Save the Lambda function and deploy it.
2.	Go to the Lambda Console and click Test.
3.	Configure a test event (use the default template; the function does not rely on event data for this task).
4.	Run the test and check the Logs in CloudWatch for the function output.
________________________________________
Step 5: Verify the Results
1.	Go to the EC2 Dashboard.
2.	Confirm:
o	Instances tagged Auto-Stop should now be stopped.
o	Instances tagged Auto-Start should now be running.
________________________________________
Valuation Criteria Addressed
•	Correctness of Code: The script uses Boto3 effectively to filter, stop, and start EC2 instances based on tags.
•	Task Completion: By manually invoking the function and verifying the instance states, the task is fully completed.
________________________________________
This step-by-step guide ensures you meet all requirements for the assignment. Let me know if you need further clarification!



Question 3:- 

Verify Bucket Encryption
1.	Check Encryption for a Bucket:
o	Navigate to the S3 Console.
o	Select the bucket you created.
o	Go to the Properties tab.
o	Under Default encryption, confirm:
	No encryption is listed for unencrypted buckets.
	AES-256 or AWS-KMS is listed for encrypted buckets.
2.	Check Encryption Using AWS CLI: Use the following command to check encryption for a bucket:
bash
Copy code
aws s3api get-bucket-encryption --bucket <bucket-name>
o	For unencrypted buckets, you will see:
json
Copy code
An error occurred (ServerSideEncryptionConfigurationNotFoundError) when calling the GetBucketEncryption operation: The server-side encryption configuration was not found
o	For encrypted buckets, you will see:
json
Copy code
{
    "ServerSideEncryptionConfiguration": {
        "Rules": [
            {
                "ApplyServerSideEncryptionByDefault": {
                    "SSEAlgorithm": "AES256"
                }
            }
        ]
    }
}
________________________________________
Key Notes
•	Unencrypted Buckets:
o	By default, S3 buckets are created without server-side encryption unless explicitly enabled.
•	Encrypted Buckets:
o	Always apply server-side encryption in production environments to protect sensitive data.
•	AWS CLI to Disable Encryption: If a bucket has encryption enabled, you can disable it using the following command:
bash
Copy code
aws s3api delete-bucket-encryption --bucket <bucket-name>
________________________________________
Let me know if you need further assistance!
4o
You said:
 
where is encrption mentioned. Choose the Region where the bucket will be created (e.g., us-east-1). Skip Server-Side Encryption: In the Bucket settings for server-side encryption, uncheck or leave the "Default encryption" setting disabled. By default, server-side encryption is not applied unless explicitly configured.
ChatGPT said:
ChatGPT
In the screenshot, the Default encryption section is clearly visible. Here's how you can configure it:
Steps to Disable Server-Side Encryption:
1.	Navigate to the "Default Encryption" Section:
o	Scroll down in the bucket creation wizard until you see the Default encryption section.
2.	Disable Encryption:
o	Ensure that none of the encryption options (SSE-S3, SSE-KMS, or DSSE-KMS) are selected or enabled.
o	If Server-side encryption is checked or selected, uncheck or leave it disabled.
3.	Bucket Key Setting:
o	Ensure that the Bucket Key option remains irrelevant (disabled) as it applies only when encryption is enabled.
4.	Complete the Configuration:
o	Leave the encryption type blank, which will create the bucket without server-side encryption.
o	Proceed by clicking the Create Bucket button to finalize.
________________________________________
Outcome:
This ensures the bucket is created without any server-side encryption by default. You can verify the absence of encryption later by checking the Bucket properties under the S3 Dashboard.
If you encounter any issues, let me know!
4o
You said:
 
ChatGPT said:
ChatGPT
In the screenshot, you can clearly see the Default encryption section. To create a bucket without server-side encryption, you need to ensure no encryption type is applied.
Steps to Create an S3 Bucket Without Server-Side Encryption
1.	Go to "Default Encryption":
o	Scroll to the Default encryption section.
2.	Disable Encryption:
o	Ensure none of the encryption types (SSE-S3, SSE-KMS, or DSSE-KMS) are selected.
o	Leave the Default encryption setting blank or unchecked.
3.	Bucket Key Option:
o	Ignore the "Bucket Key" option since it applies only when encryption is enabled.
4.	Complete Bucket Setup:
o	Scroll down to the Advanced settings section and adjust additional settings as needed.
o	Click the Create bucket button to finalize the bucket creation.
________________________________________
Verification
•	After creating the bucket, navigate to the Properties tab of the bucket in the S3 Dashboard.
•	Under the Default encryption section, it should say "None," confirming the bucket does not have server-side encryption.
Let me know if you need further clarification!
4o
You said:
 
ism this correct 
ChatGPT said:
ChatGPT
The screenshot shows the Default encryption section, but it is not correct for creating a bucket without server-side encryption because one of the encryption options (Server-side encryption with Amazon S3 managed keys (SSE-S3)) is selected.
To Create a Bucket Without Server-Side Encryption:
1.	Uncheck All Encryption Options:
o	Ensure no encryption type (SSE-S3, SSE-KMS, or DSSE-KMS) is selected.
o	There should be no selection in the Encryption type section.
2.	Disable Bucket Key:
o	The Bucket Key option is irrelevant for buckets without encryption, but leaving it as Disable is fine.
3.	Save Settings:
o	Scroll down and complete the bucket creation process.
________________________________________
How to Verify the Bucket is Not Encrypted:
1.	In the S3 Console:
o	Go to the Properties tab of the bucket.
o	In the Default encryption section, it should display None.
2.	Using AWS CLI: Run the following command:
bash
Copy code
aws s3api get-bucket-encryption --bucket <bucket-name>
o	If no encryption is applied, you will see an error like this:
json
Copy code
An error occurred (ServerSideEncryptionConfigurationNotFoundError) when calling the GetBucketEncryption operation: The server-side encryption configuration was not found
________________________________________
Action to Correct:
Deselect the encryption type and create the bucket again to ensure no server-side encryption is applied. Let me know if you need further clarification!
4o



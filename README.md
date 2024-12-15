

Assignment 18: Autosave EC2 Instance State Before Shutdown
Objective: Before an EC2 instance is shut down, automatically save its current state to an S3 bucket.
Instructions:
1. Create a Lambda function.
2. Using Boto3, the function should:
   1. Detect when an EC2 instance is about to be terminated.
   2. Save the current state or necessary files from the EC2 instance to an S3 bucket.
3. Use CloudWatch Events to trigger this Lambda function whenever an EC2 termination command is detected.

# Documentation: Autosave EC2 Instance State Before Shutdown
---
## Objective
The goal of this assignment is to create a process where, before an EC2 instance is shut down or terminated, its current state is automatically saved to an S3 bucket. This document outlines the steps, configuration, and troubleshooting performed to achieve this functionality.
---
## Solution Overview

1. **CloudWatch Rule**: Detect EC2 instance state-change events such as `shutting-down` or `terminated`.
2. **Lambda Function**: Process the events, fetch the EC2 instance details, and store them in an S3 bucket.
3. **S3 Bucket**: Store instance details as JSON files.
4. **IAM Roles and Policies**: Configure permissions for Lambda and S3 access.
5. **Testing and Troubleshooting**: Validate functionality and resolve issues.
---
## Step-by-Step Implementation
-----------------------------------------------------------------------------
### Step 1: Create an S3 Bucket
1. Navigate to the **S3 Console**.
2. Click **"Create Bucket"**.
3. Enter a unique bucket name, such as `ec2-instance-state-backup`.
4. Leave default settings (Block Public Access enabled) and click **"Create Bucket"**.
5. Set the bucket policy to allow access from the Lambda function:
   ```json
   {
       "Version": "2012-10-17",
       "Statement": [
           {
               "Effect": "Allow",
               "Principal": {
                   "AWS": "arn:aws:iam::225989348530:role/service-role/AutoTagEC2Instance-role-35hgkdnr"
               },
               "Action": "s3:PutObject",
               "Resource": "arn:aws:s3:::ec2-instance-state-backup/*"
           }
       ]
   }
   ```
----------------------------------------------------- 
### Step 2: Configure IAM Role for Lambda
1. Navigate to the **IAM Console**.
2. Create a role for **AWS Lambda**.
3. Attach the following policies to the role:
   - **AmazonEC2ReadOnlyAccess** (to read instance details).
   - **AmazonS3FullAccess** (or a custom policy with `s3:PutObject` permissions for the S3 bucket).

   Custom policy for S3:
   ```json
   {
       "Version": "2012-10-17",
       "Statement": [
           {
               "Effect": "Allow",
               "Action": "s3:PutObject",
               "Resource": "arn:aws:s3:::ec2-instance-state-backup/*"
           }
       ]
   }
-------------------------------------------   ```
4. Assign the role to your Lambda function.
- ![image](https://github.com/user-attachments/assets/08918c3b-ef10-401d-9f9d-50b881300dbe)
- ![image](https://github.com/user-attachments/assets/fc6c0f39-87df-4978-8716-afc720791869)


### Step 3: Create a CloudWatch Rule
1. Navigate to the **CloudWatch Console**.
2. Go to **Rules** and click **"Create Rule"**.
3. Set the event source:
   - **Service Name**: `EC2`
   - **Event Type**: `EC2 Instance State-change Notification`
   - **Specific State**: `shutting-down`
4. Add the Lambda function as the target.
5. Name the rule `ec2-shutdown-trigger` and enable it.
- ![image](https://github.com/user-attachments/assets/425783a3-fac4-43d9-94e2-aca7fb10cfa6)
- ![image](https://github.com/user-attachments/assets/5c50d688-f138-4289-9c97-759303d23372)

-----------------------------------------
### Step 4: Write the Lambda Function

Deploy the following Python code as your Lambda function:

```python
import boto3
import json
from datetime import datetime

# Helper function to convert datetime to string
def json_serial(obj):
    """JSON serializer for objects not serializable by default"""
    if isinstance(obj, datetime):
        return obj.isoformat()
    raise TypeError("Type not serializable")

def lambda_handler(event, context):
    print("Received event:", json.dumps(event))  # Log the incoming event
    
    try:
        # Validate event structure
        if 'detail' not in event or 'instance-id' not in event['detail']:
            print("Invalid event structure. 'detail' or 'instance-id' not found.")
            return {
                "statusCode": 400,
                "body": json.dumps("Invalid event structure")
            }
        
        # Extract instance ID and state
        instance_id = event['detail']['instance-id']
        state = event['detail']['state']
        print(f"Instance ID: {instance_id}, State: {state}")
        
        # Only process if the state is 'shutting-down' or 'terminated'
        if state not in ['shutting-down', 'terminated']:
            print(f"Instance {instance_id} is in state '{state}', skipping processing.")
            return {
                "statusCode": 200,
                "body": json.dumps(f"Instance {instance_id} is in state '{state}', no action taken.")
            }
        
        # Initialize AWS clients
        ec2_client = boto3.client('ec2')
        s3_client = boto3.client('s3')
        bucket_name = 'ec2-instance-state-backup'

        # Get instance details
        response = ec2_client.describe_instances(InstanceIds=[instance_id])
        instance_state = response['Reservations'][0]['Instances'][0]
        print(f"Instance state: {json.dumps(instance_state, indent=4, default=json_serial)}")
        
        # Save instance state to S3
        s3_key = f"{instance_id}-{state}.json"
        s3_client.put_object(
            Bucket=bucket_name,
            Key=s3_key,
            Body=json.dumps(instance_state, indent=4, default=json_serial)
        )
        print(f"Successfully saved instance state to S3: {bucket_name}/{s3_key}")
        
        return {
            "statusCode": 200,
            "body": json.dumps(f"State saved for instance {instance_id}")
        }

    except Exception as e:
        print(f"Error: {str(e)}")  # Log any unexpected errors
        return {
            "statusCode": 500,
            "body": json.dumps(f"Error: {str(e)}")
        }

------------
### Step 5: Testing
1. **Simulate a Shutdown Event**:
   - Terminate or shut down an EC2 instance.
2. **Validate S3**:
   - Go to the S3 bucket (`ec2-instance-state-backup`) and verify that a file named `<instance-id>-shutting-down.json` or `<instance-id>-terminated.json` is created.
   - ![image](https://github.com/user-attachments/assets/8b334a24-0cb8-4f12-90ee-5e4802cf9646)
   - ![image](https://github.com/user-attachments/assets/4f9152fc-b4be-47a0-9e19-6afdb44fd0b4)
------------
### Step 6: Troubleshooting
1. **Error: Object of type datetime is not JSON serializable**:
   - Resolved by adding the `json_serial` helper function to convert `datetime` objects to ISO 8601 string format.
2. **Error: Invalid event structure**:
   - Verified the CloudWatch Rule to ensure it sends the correct payload.
3. **S3 Upload Issues**:
   - Ensured the Lambda role had `s3:PutObject` permissions.
   - Verified the S3 bucket policy was correctly configured.
4. **Manual Testing**:
   - Used a simulated event payload to test the Lambda function in the AWS Console:
     ```json
     {
         "version": "0",
         "id": "cd2d7024-22b4-4a35-91c9-ef5b88b43326",
         "detail-type": "EC2 Instance State-change Notification",
         "source": "aws.ec2",
         "account": "225989348530",
         "time": "2024-12-15T00:12:43Z",
         "region": "us-east-1",
         "resources": ["arn:aws:ec2:us-east-1:225989348530:instance/i-0b48e6081f8209438"],
         "detail": {
           "instance-id": "i-0b48e6081f8209438",
           "state": "shutting-down"
         }
     }
     ```
- ![image](https://github.com/user-attachments/assets/cabe16ae-2a80-4f83-94e5-7afe791bbae0)




## Conclusion
This solution ensures that EC2 instance details are automatically saved to an S3 bucket before termination or shutdown, providing a reliable backup mechanism. By leveraging AWS services such as CloudWatch, Lambda, and S3, the process is fully automated and scalable.
---


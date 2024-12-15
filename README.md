



























































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
====================================================================================================================================================================================================================================================================


Assignment 5: Auto-Tagging EC2 Instances on Launch Using AWS Lambda and Boto3
Objective: Learn to automate the tagging of EC2 instances as soon as they are launched, ensuring better resource tracking and management.
Task: Automatically tag any newly launched EC2 instance with the current date and a custom tag.
Instructions:
1. EC2 Setup:
   - Ensure you have the capability to launch EC2 instances.
2. Lambda IAM Role:
   - In the IAM dashboard, create a new role for Lambda.
   - Attach the `AmazonEC2FullAccess` policy to this role.
3. Lambda Function:
   - Navigate to the Lambda dashboard and create a new function.
   - Choose Python 3.x as the runtime.
   - Assign the IAM role created in the previous step.
   - Write the Boto3 Python script to:
     1. Initialize a boto3 EC2 client.
     2. Retrieve the instance ID from the event.
     3. Tag the new instance with the current date and another tag of your choice.
    4. Print a confirmation message for logging purposes.
4. CloudWatch Events:
   - Set up a CloudWatch Event Rule to trigger the EC2 instance launch event.
   - Attach the Lambda function as the target.
5. Testing:
   - Launch a new EC2 instance.
   - After a short delay, confirm that the instance is automatically tagged as specified.

----------------------------------------------

# Assignment: Auto-Tagging EC2 Instances on Launch Using AWS Lambda and Boto3

## Objective
To automate the tagging of EC2 instances as soon as they are launched, ensuring better resource tracking and management.

---
## Steps

### Step 1: Prerequisites
- Ensure you have:
  - An active AWS account.
  - Access to **Amazon EC2**, **IAM**, **Lambda**, **EventBridge**, and **CloudWatch Logs** services.
 
--------------------------------------------------------------------------------

### Step 2: Create an IAM Role for Lambda

1. **Go to the IAM Console:**
   - Open the AWS Management Console and navigate to **IAM**.

2. **Create a New Role:**
   - Click **Roles** → **Create Role**.
   - Select **AWS Service** → **Lambda** → **Next**.

3. **Attach Permissions:**
   - Search for and attach the following policies:
     - **AmazonEC2FullAccess** (or a custom policy with `ec2:CreateTags` permission).

4. **Review and Create the Role:**
   - Name the role (e.g., `AutoTagEC2InstanceRole`) and click **Create Role**.
   - ![image](https://github.com/user-attachments/assets/e33c57bf-acc4-4b19-b870-a5101f85e3c2)
   - ![image](https://github.com/user-attachments/assets/c52fe561-de23-4eba-80a3-e73cd1a43796)

---------------------------------------------------------------------------------------------------------

### Step 3: Create the Lambda Function

1. **Go to the Lambda Console:**
   - Open the AWS Management Console and navigate to **Lambda**.

2. **Create a New Function:**
   - Click **Create Function** → Select **Author from Scratch**.
   - Provide a name (e.g., `AutoTagEC2Instance`).
   - Set the runtime to **Python 3.x**.
   - Select the IAM role created earlier (`AutoTagEC2InstanceRole`).
   - Click **Create Function**.

3. **Add the Lambda Code:**
   - Replace the default code with the following:

     ```python
     import json
     import boto3
     import datetime

     def lambda_handler(event, context):
         # Initialize EC2 client
         ec2_client = boto3.client('ec2')

         try:
             # Log the incoming event for debugging
             print("Received event:", json.dumps(event, indent=2))

             # Extract the instance ID from the event
             if 'detail' in event and 'instance-id' in event['detail']:
                 instance_id = event['detail']['instance-id']
             else:
                 raise KeyError("The event does not contain the required 'detail' or 'instance-id' keys.")

             # Get the current date
             current_date = datetime.datetime.now().strftime('%Y-%m-%d')

             # Define the tags
             tags = [
                 {'Key': 'LaunchDate', 'Value': current_date},
                 {'Key': 'Owner', 'Value': 'AutoTagging'}
             ]

             # Apply tags to the EC2 instance
             response = ec2_client.create_tags(Resources=[instance_id], Tags=tags)

             # Log the success
             print(f"Successfully tagged instance {instance_id} with tags: {tags}")

             return {
                 'statusCode': 200,
                 'body': f"Tags applied successfully to instance {instance_id}: {tags}"
             }

         except KeyError as e:
             # Handle missing keys in the event
             print(f"KeyError: {str(e)}")
             return {
                 'statusCode': 400,
                 'body': f"Invalid event structure: {str(e)}"
             }

         except Exception as e:
             # Catch all other exceptions
             print(f"Unexpected error: {str(e)}")
             return {
                 'statusCode': 500,
                 'body': f"Failed to apply tags: {str(e)}"
             }
     ```
4. **Deploy the Code:**
   - Click **Deploy** to save the function.

---------------------------------------------------------------------------------------------------------------------------------------------------
### Step 4: Set Up an EventBridge Rule

1. **Go to the EventBridge Console:**
   - Navigate to **Amazon EventBridge**.

2. **Create a New Rule:**
   - Click **Create Rule** → Name the rule (e.g., `TagEC2OnLaunch`).

3. **Set the Event Pattern:**
   - Select **Event source** → **AWS services** → **EC2**.
   - Under **Event type**, choose **EC2 Instance State-change Notification**.
   - Add a filter for `state` → `running` in the **Event Pattern** section.

   Final Event Pattern JSON:
   ```json
   {
       "source": ["aws.ec2"],
       "detail-type": ["EC2 Instance State-change Notification"],
       "detail": {
           "state": ["running"]
       }
}
   - ![image](https://github.com/user-attachments/assets/12853eb6-f374-4131-ade9-02a41d36afa8)

   - 4. **Set the Target:**
   - Set the target to the Lambda function created earlier (`AutoTagEC2Instance`).

5. **Enable the Rule:**
   - Save and enable the rule.

---------------------------------------------------------------------------------------------------

### Step 5: Testing

#### Manually Test Lambda:
1. Go to the **Test** tab in Lambda.
2. Create a new test event with the following structure:
   ```json
   {
       "version": "0",
       "id": "abcd1234-5678-90ab-cdef-EXAMPLE",
       "detail-type": "EC2 Instance State-change Notification",
       "source": "aws.ec2",
       "account": "123456789012",
       "time": "2024-12-14T12:34:56Z",
       "region": "us-east-1",
       "resources": ["arn:aws:ec2:us-east-1:123456789012:instance/i-0bcf4a6cdc4d9fe57"],
       "detail": {
           "instance-id": "i-0bcf4a6cdc4d9fe57",
           "state": "running"
       }
   }
   ```
3. Replace `i-0bcf4a6cdc4d9fe57` with a valid instance ID.
4. Run the test and check the logs in **CloudWatch Logs** to verify the tags were applied.

#### Dynamic Testing via EventBridge:
1. Launch a new EC2 instance:
   - Go to the **EC2 Dashboard** → Click **Launch Instance**.
   - Use default settings (e.g., Amazon Linux 2, `t2.micro`).
2. Verify that:
   - The EventBridge rule triggers the Lambda function.
   - Tags (`LaunchDate` and `Owner`) are applied to the newly launched instance.
     output :- 
   - ![image](https://github.com/user-attachments/assets/70aa31f5-5198-4aa2-a213-12fb27670c12)

### Step 6: Clean Up Resources

To avoid incurring unnecessary charges, terminate the resources:
1. **Terminate EC2 Instances:**
   - Go to the EC2 Dashboard → Select and terminate any test instances.
2. **Delete the Lambda Function and EventBridge Rule (if not needed):**
   - Navigate to the **Lambda Console** → Delete the function.
   - Go to the **EventBridge Console** → Delete the rule.

-------------------------------------------------------------------------------------------------------------------------------------------------------

## Troubleshooting

### Issue: `KeyError: 'detail'`
- **Cause:** The event structure passed to the Lambda function does not contain the required `detail` or `instance-id` keys.
- **Solution:**
  1. Verify the EventBridge rule's event pattern matches the correct structure.
  2. Test the Lambda function with a valid event JSON containing `detail` and `instance-id` keys.

### Issue: `An error occurred (InvalidID) when calling the CreateTags operation`
- **Cause:** The `instance-id` provided in the event is invalid or not found in the account.
- **Solution:**
  1. Ensure the `instance-id` in the test event corresponds to a valid EC2 instance in your account.
  2. Use dynamic testing by launching a new EC2 instance to trigger the EventBridge rule.

### Issue: `An error occurred (UnauthorizedOperation)`
- **Cause:** The IAM role attached to the Lambda function does not have sufficient permissions to create tags.
- **Solution:**
  1. Attach the **AmazonEC2FullAccess** policy to the IAM role.
  2. Alternatively, create a custom policy with the `ec2:CreateTags` action.

------------------------------------------------------------------------------------------------------------------------------

## Commands for Reference

### Boto3 Create Tags Command (Python):
```python
response = ec2_client.create_tags(Resources=[instance_id], Tags=tags)
```

### AWS CLI to Tag an Instance Manually (Optional):
```bash
aws ec2 create-tags --resources i-0bcf4a6cdc4d9fe57 --tags Key=LaunchDate,Value=2024-12-14 Key=Owner,Value=AutoTagging
=========================================================================================================================================================================================================================================================








# Assignment 1: Automated Instance Management Using AWS Lambda and Boto3

## Objective
The objective of this assignment is to:

- Gain hands-on experience with AWS Lambda and Boto3 for cloud automation.
- Automatically manage EC2 instances by starting or stopping them based on their tags.
## Task
You are asked with automating the management of EC2 instances using AWS Lambda and Python's Boto3 library:
- Stop instances tagged as `Auto-Stop`.
- Start instances tagged as `Auto-Start`.
## Specific Requirements
1. **Setup**
   - Launch two EC2 instances in AWS.
   - Assign the following tags:

     **Instance 1:**
     - **Key:** Action
     - **Value:** Auto-Stop

     **Instance 2:**
     - **Key:** Action
     - **Value:** Auto-Start

2. **Lambda Function**
   - Create a Lambda function that:
     - Describes EC2 instances using Boto3.
     - Identifies instances tagged as `Auto-Stop` and stops them.
     - Identifies instances tagged as `Auto-Start` and starts them.

3. **Testing**
   - Manually invoke the Lambda function and verify:
     - Instances tagged as `Auto-Stop` are stopped.
     - Instances tagged as `Auto-Start` are started.


## Documentation :- Automated Instance Management Using AWS Lambda and Boto3


## Step-by-Step Implementation

### Step 1: Create EC2 Instances

1. **Navigate to the EC2 Dashboard:**
   - Open the AWS Console and search for `EC2`.

2. **Launch Instances:**
   - Click **Launch Instance**.
   - Select a free-tier eligible instance type (e.g., `t2.micro`).

3. **Assign Tags:**
   - While configuring the instance, add the following tags:

     **Instance 1:**
     - **Key:** Action
     - **Value:** Auto-Stop

     **Instance 2:**
     - **Key:** Action
     - **Value:** Auto-Start
 -  ![image](https://github.com/user-attachments/assets/816ae231-577a-4e7a-84c9-f6768158f47e)
 -  ![image](https://github.com/user-attachments/assets/22bac472-d88a-412f-b068-4c59f53a526e)


4. **Verify Tags:**
   - Go to the **Tags** tab of each instance in the EC2 Dashboard to confirm the correct key-value pairs.

---

### Step 2: Configure an IAM Role for Lambda

1. **Go to the IAM Console:**
   - Navigate to **Roles** → **Create Role**.

2. **Choose Trusted Entity:**
   - Select **AWS Service** and choose **Lambda**.

3. **Attach Policy:**
   - For simplicity during testing, attach the `AmazonEC2FullAccess` policy.
   - For better security, use the following custom policy:

   ```json
   {
       "Version": "2012-10-17",
       "Statement": [
           {
               "Effect": "Allow",
               "Action": [
                   "ec2:DescribeInstances",
                   "ec2:StartInstances",
                   "ec2:StopInstances"
               ],
               "Resource": "*"
           }
       ]
   }
   ```
   - ![image](https://github.com/user-attachments/assets/68cdf254-b43e-4f44-8786-9d3a056975bd)


4. **Name the Role:**
   - Name it `Ec2tagLambda` and save it.

5. **Attach the Role to Lambda:**
   - This role will allow the Lambda function to manage EC2 instances.

---

### Step 3: Create the Lambda Function

1. **Navigate to the AWS Lambda Console:**
   - Select **Create Function**.

2. **Set Configuration:**
   - **Name:** EC2InstanceManager
   - **Runtime:** Python 3.x
   - **Execution Role:** Attach the `Lambda_EC2_Manager_Role`.

3. **Paste the Code:**
   - Use the following Python script in the function editor.
   - ![image](https://github.com/user-attachments/assets/e41e1768-0605-4164-a9fe-1493c8726c96)


#### Lambda Function Code

```python
import boto3
import logging

# Set up logging
logger = logging.getLogger()
logger.setLevel(logging.INFO)

def lambda_handler(event, context):
    ec2 = boto3.client('ec2')
    
    # Describe instances with Action tags
    response = ec2.describe_instances(Filters=[
        {'Name': 'tag:Action', 'Values': ['Auto-Stop', 'Auto-Start']}
    ])
    
    auto_stop_instances = []
    auto_start_instances = []

    # Process the response
    for reservation in response['Reservations']:
        for instance in reservation['Instances']:
            instance_id = instance['InstanceId']
            state = instance['State']['Name']
            for tag in instance.get('Tags', []):
                if tag['Key'] == 'Action':
                    if tag['Value'] == 'Auto-Stop' and state == 'running':
                        auto_stop_instances.append(instance_id)
                    elif tag['Value'] == 'Auto-Start' and state == 'stopped':
                        auto_start_instances.append(instance_id)
    
    # Stop instances
    if auto_stop_instances:
        logger.info(f"Stopping instances: {auto_stop_instances}")
        ec2.stop_instances(InstanceIds=auto_stop_instances)
    else:
        logger.info("No 'Auto-Stop' instances found.")
    
    # Start instances
    if auto_start_instances:
        logger.info(f"Starting instances: {auto_start_instances}")
        ec2.start_instances(InstanceIds=auto_start_instances)
    else:
        logger.info("No 'Auto-Start' instances found.")
    
    return {
        "statusCode": 200,
        "body": "Instance management completed successfully!"
    }
```

---

### Step 4: Test the Lambda Function

1. **Create a Test Event:**
   - Use a simple JSON payload like this:

     ```json
     {}
     ```

2. **Run the Function:**
   - Click **Test** in the Lambda Console.

3. **Verify Logs:**
   - Go to the **Monitor** tab → **CloudWatch Logs**.
   - Confirm that instances with `Auto-Stop` tags were stopped and `Auto-Start` tags were started.

---
- ![image](https://github.com/user-attachments/assets/aecbb0ff-0748-4420-8171-4213ab579da1)



### Step 5: Verify in EC2

1. **Go to EC2 Dashboard:**
   - Check the **Instance State**.

2. **Expected Outcome:**
   - Instances with `Auto-Stop` → Stopped.
   - Instances with `Auto-Start` → Running.

- ![image](https://github.com/user-attachments/assets/5559d030-47ea-42a9-b193-993a1084d6e8)


## Troubleshooting

### 1. IAM Permissions Issue

- **Error:** Access Denied.
- **Solution:** Ensure the IAM role attached to the Lambda function has permissions for:
  - `ec2:DescribeInstances`
  - `ec2:StartInstances`
  - `ec2:StopInstances`.

### 2. Tag Mismatch

- **Issue:** Lambda function does not detect instances.
- **Solution:**
  - Ensure tags are:
    - **Key:** Action
    - **Value:** Auto-Stop or Auto-Start.
  - Tags are case-sensitive.

### 3. Logs Show No Action

- **Cause:** Instances are already in the desired state.
- **Solution:** Confirm:
  - `Auto-Stop` instances are running.
  - `Auto-Start` instances are stopped.

---

## Deliverables

1. **Screenshots:**
   - Before and after Lambda execution, showing instance states.

2. **Lambda Code:**
   - Include the Python script.

3. **Summary Report:**
   - Document the steps, challenges, and solutions.

---
## Conclusion

By completing this assignment, you learned:

- How to automate EC2 management using AWS Lambda and Boto3.
- How to configure IAM roles and manage permissions effectively.
- How to debug and troubleshoot Lambda functions.


==========================================================================================================================================================================================
======================================================================================================================================================================

# **ASSIGNMENT 18: AUTOSAVE EC2 INSTANCE STATE BEFORE SHUTDOWN**

### **Objective**
Before an EC2 instance is shut down, automatically save its current state to an S3 bucket.

### **Instructions**
1. **Create a Lambda Function**.
2. Using **Boto3**, the function should:
   - Detect when an EC2 instance is about to be terminated.
   - Save the current state or necessary files from the EC2 instance to an S3 bucket.
3. Use **CloudWatch Events** to trigger this Lambda function whenever an EC2 termination command is detected.

---

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
5. - ![image](https://github.com/user-attachments/assets/70259562-766b-4b71-8a44-db4fdd748e28)

6. Set the bucket policy to allow access from the Lambda function:
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
   - ![image](https://github.com/user-attachments/assets/705bc7c4-0bcb-45e8-97a7-70f3a9470fb8)


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
   ```
   
4. Assign the role to your Lambda function.

### Step 3: Create a CloudWatch Rule
1. Navigate to the **CloudWatch Console**.
2. Go to **Rules** and click **"Create Rule"**.
3. Set the event source:
   - **Service Name**: `EC2`
   - **Event Type**: `EC2 Instance State-change Notification`
   - **Specific State**: `shutting-down`
4. Add the Lambda function as the target.
5. Name the rule `ec2-shutdown-trigger` and enable it.
 - ![image](https://github.com/user-attachments/assets/2ab9b12c-ac57-4cc4-84ba-667ee608ab77)

-

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
```

------------
### Step 5: Testing
1. **Simulate a Shutdown Event**:
   - Terminate or shut down an EC2 instance.
2. **Validate S3**:
   - Go to the S3 bucket (`ec2-instance-state-backup`) and verify that a file named `<instance-id>-shutting-down.json` or `<instance-id>-terminated.json` is created.
   - ![image](https://github.com/user-attachments/assets/29c45103-55fc-4bb9-b33d-b0538bd9c101)
    - ![image](https://github.com/user-attachments/assets/bc967801-b2a0-4cec-ab0a-aa2ccfdfc436)


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
     
## Conclusion
This solution ensures that EC2 instance details are automatically saved to an S3 bucket before termination or shutdown, providing a reliable backup mechanism. By leveraging AWS services such as CloudWatch, Lambda, and S3, the process is fully automated and scalable.
====================================================================================================================================================================================================================================================================

## **Assignment 14: Monitor EC2 Instance State Changes Using AWS Lambda, Boto3, and SNS**

### **Objective**
**Automatically monitor changes in EC2 instance states and send notifications via SNS whenever an instance is started or stopped.**

## **Task**
Set up a Lambda function that listens to EC2 state change events and sends SNS notifications detailing the state changes.

-## **Instructions**

#### **1. SNS Setup:**
- Navigate to the SNS dashboard and create a new topic.
- Subscribe to this topic with your email.

#### **2. Lambda IAM Role:**
- Create a role with permissions to read EC2 instance states and send SNS notifications.

#### **3. Lambda Function:**
- Create a function and assign the above IAM role.
- Use Boto3 to:
  1. Extract details from the event regarding the EC2 state change.
  2. Send an SNS notification with details about which EC2 instance changed state and the new state (e.g., started, stopped).

#### **4. EC2 Event Bridge (formerly CloudWatch Events):**
- Set up an Event Bridge rule to trigger your Lambda function whenever an EC2 instance state changes.

#### **5. Testing:**
- Start or stop one of your EC2 instances.
- Confirm you receive an SNS notification about the state change.

 Documentation:- Monitoring EC2 Instance State Changes Using AWS Lambda, Boto3, and SNS

## **Step 1: SNS Setup**

### **1.1 Create an SNS Topic**
1. Log in to the AWS Management Console and navigate to **SNS (Simple Notification Service)**.
2. Click **Topics** in the left-hand menu and then click **Create topic**.
3. Select **Standard** as the type.
4. Enter a name for your topic, e.g., `EC2StateChangeTopic`.
5. Click **Create topic**.

### **1.2 Subscribe to the Topic**
1. Open your newly created topic.
2. Click **Create subscription**.
3. Choose `Email` as the protocol.
4. Enter your email address in the **Endpoint** field.
5. Click **Create subscription**.
6. Check your email inbox for a confirmation email from AWS and confirm your subscription.

---

## **Step 2: Create an IAM Role for Lambda**

### **2.1 Create a Role**
1. Navigate to the **IAM (Identity and Access Management)** dashboard.
2. Click **Roles** in the left-hand menu and then click **Create role**.
3. Select **AWS service** as the trusted entity type.
4. Choose **Lambda** as the service.
5. Click **Next** to add permissions.

### **2.2 Attach Policies**
1. Search for and attach the following policies:
   - **AmazonEC2ReadOnlyAccess** (to allow Lambda to read EC2 instance states).
   - **AmazonSNSFullAccess** (to allow Lambda to publish messages to SNS).

2. Alternatively, create a custom policy using the following JSON and attach it:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "ec2:DescribeInstances",
                "ec2:DescribeInstanceStatus"
            ],
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": "sns:Publish",
            "Resource": "<YOUR_SNS_TOPIC_ARN>"
        }
    ]
}
```

3. Click **Next**, give your role a name, e.g., `LambdaEC2StateChangeRole`.
4. Click **Create role**.
5. 

---

## **Step 3: Create the Lambda Function**

### **3.1 Create a Lambda Function**
1. Go to the **Lambda dashboard** in AWS.
2. Click **Create function**.
3. Choose **Author from scratch**.
4. Enter a name for the function, e.g., `EC2StateChangeNotifier`.
5. Select **Python 3.x** as the runtime.
6. Under **Execution role**, choose **Use an existing role** and select the role you created in Step 2.
7. Click **Create function**.
   -


### **3.2 Add the Code**
1. In the Lambda code editor, replace the default code with the following:

```python
import json
import boto3

def lambda_handler(event, context):
    sns_client = boto3.client('sns')
    sns_topic_arn = '<YOUR_SNS_TOPIC_ARN>'

    # Extract details from the event
    detail = event['detail']
    instance_id = detail['instance-id']
    state = detail['state']

    # Prepare the message
    message = f"EC2 Instance {instance_id} is now {state}."
    print(message)

    # Publish the message to SNS
    response = sns_client.publish(
        TopicArn=sns_topic_arn,
        Message=message,
        Subject='EC2 State Change Notification'
    )
    print("SNS Response:", response)
```

2. Replace `<YOUR_SNS_TOPIC_ARN>` with the ARN of your SNS topic (found in the SNS dashboard).
3. Click **Deploy** to save your changes.

---

## **Step 4: Configure EventBridge to Trigger Lambda**

### **4.1 Create an EventBridge Rule**
1. Navigate to **EventBridge** in the AWS Management Console.
2. Click **Rules** in the left-hand menu and then click **Create rule**.
3. Enter a name for the rule, e.g., `EC2StateChangeRule`.
4. For **Event source**, choose **AWS services**.
5. Under **Service Name**, select **EC2**.
6. For **Event Type**, select **EC2 Instance State-change Notification**.
   


### **4.2 Add Target**
1. Under the **Target** section, choose **Lambda function**.
2. Select your Lambda function (`EC2StateChangeNotifier`).
3. Click **Create**.
4


---

## **Step 5: Testing**

### **5.1 Start or Stop an EC2 Instance**
1. Navigate to the **EC2 dashboard**.
2. Select an instance, then choose **Instance State** > **Start** or **Stop**.
3. Wait for the instance state to change.

### **5.2 Verify SNS Notification**
1. Check your email inbox.
2. You should receive a notification about the instance state change, e.g., `EC2 Instance i-0123456789abcdef is now stopped`.



---



====================================================================================================================================================================================================================================================================

# Assignment 5: Auto-Tagging EC2 Instances on Launch Using AWS Lambda and Boto3

## Objective
Learn to automate the tagging of EC2 instances as soon as they are launched, ensuring better resource tracking and management.

## Task
Automatically tag any newly launched EC2 instance with the current date and a custom tag.

---


### 3. Lambda Function
- Navigate to the **Lambda** dashboard and create a new function.
- Choose **Python 3.x** as the runtime.
- Assign the IAM role created in the previous step.
- 
- Write the Boto3 Python script to:
  1. Initialize a boto3 EC2 client.
  2. Retrieve the instance ID from the event.
  3. Tag the new instance with the current date and another tag of your choice.
  4. Print a confirmation message for logging purposes.

### 4. CloudWatch Events
- Set up a CloudWatch Event Rule to trigger the EC2 instance launch event.
- Attach the Lambda function as the target.

### 5. Testing
- Launch a new EC2 instance.
- After a short delay, confirm that the instance is automatically tagged as specified.


## Instructions

### 1. EC2 Setup
- Ensure you have the capability to launch EC2 instances.

### 2. Lambda IAM Role
- In the **IAM** dashboard, create a new role for **Lambda**.
- Attach the `AmazonEC2FullAccess` policy to this role.
- ![image](https://github.com/user-attachments/assets/e33c57bf-acc4-4b19-b870-a5101f85e3c2)
- ![image](https://github.com/user-attachments/assets/c52fe561-de23-4eba-80a3-e73cd1a43796)

  #### 3. Lambda Function
1. Navigate to the **Lambda** dashboard and create a new function.
2. Choose **Python 3.x** as the runtime.
3. Assign the IAM role created in the previous step.
4. Replace the default code with the following:

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

5. **Deploy** the code to save the function.
---

#### 4. CloudWatch Events
1. Go to the **EventBridge Console**.
2. **Create a New Rule:**
   - Name the rule (e.g., `TagEC2nLaunch`).
3. **Set the Event Pattern:**
   - Select **Event source** → **AWS services** → **EC2**.
   - Under **Event type**, choose **EC2 Instance State-change Notification**.
   - Add a filter for `state` → `running` in the **Event Pattern** section.

   Final Event Pattern JSON:

   json
   {
       "source": ["aws.ec2"],
       "detail-type": ["EC2 Instance State-change Notification"],
       "detail": {
           "state": ["running"]
       }
   }


   ![image](https://github.com/user-attachments/assets/12853eb6-f374-4131-ade9-02a41d36afa8)`

   
5. **Set the Target:** Attach the Lambda function created earlier (`AutoTagEC2Instance`).
6. Save and enable the rule.

---

### Testing

#### Manual Test
1. Go to the **Test** tab in Lambda.
2. Use the following test event:

json
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

3. Replace `i-0bcf4a6cdc4d9fe57` with a valid instance ID.
4. Run the test and confirm tags are applied.

#### Dynamic Test
1. Launch a new EC2 instance from the EC2 Dashboard.
2. Check the **Tags** tab for the instance to confirm the tags are applied automatically.
     output :- 
   - ![image](https://github.com/user-attachments/assets/70aa31f5-5198-4aa2-a213-12fb27670c12)
---

### Troubleshooting

#### Issue: `KeyError: 'detail'`
- **Cause:** The event structure does not contain the required `detail` or `instance-id` keys.
- **Solution:** Verify the EventBridge rule's event pattern matches the expected format.

#### Issue: `InvalidID` when calling `CreateTags`
- **Cause:** The `instance-id` in the event does not exist in your account.
- **Solution:** Use a valid instance ID and ensure the EC2 instance exists.

#### Issue: `UnauthorizedOperation`
- **Cause:** The IAM role lacks sufficient permissions to tag the instance.
- **Solution:** Attach the `AmazonEC2FullAccess` policy or a custom policy with `ec2:CreateTags` permission.

---

### Clean Up Resources
1. **Terminate EC2 Instances:**
   - Go to the **EC2 Dashboard** → Select and terminate the test instances.
2. **Delete Resources:**
   - Delete the Lambda function.
   - Delete the EventBridge rule.




======================================================================================================================================================================================================================================================================================================================================================================================================================================================================================================================================



#  Assignment 14: Monitor EC2 Instance State Changes Using AWS Lambda, Boto3, and SNS

Task: Set up a Lambda function that listens to EC2 state change events and sends SNS notifications detailing the state changes.

 Monitoring EC2 Instance State Changes Using AWS Lambda, Boto3, and SNS

## Objective
Automatically monitor changes in EC2 instance states and send notifications via SNS whenever an instance is started or stopped.

## Instructions

### 1. SNS Setup

#### 1.1 Create an SNS Topic
1. Log in to the AWS Management Console and navigate to **SNS (Simple Notification Service)**.
2. Click **Topics** in the left-hand menu and then click **Create topic**.
3. Select **Standard** as the type.
4. Enter a name for your topic, e.g., `EC2StateChangeTopic`.
5. Click **Create topic**.
- - ![image](https://github.com/user-attachments/assets/aecc0d99-3f6b-4c8d-9404-43dc9c5567b2)


#### 1.2 Subscribe to the Topic
1. Open your newly created topic.
2. Click **Create subscription**.
3. Choose `Email` as the protocol.
4. Enter your email address in the **Endpoint** field.
5. Click **Create subscription**.
6. Check your email inbox for a confirmation email from AWS and confirm your subscription.

---

### 2. Create an IAM Role for Lambda

#### 2.1 Create a Role
1. Navigate to the **IAM (Identity and Access Management)** dashboard.
2. Click **Roles** in the left-hand menu and then click **Create role**.
3. Select **AWS service** as the trusted entity type.
4. Choose **Lambda** as the service.
5. Click **Next** to add permissions.
6. - - ![image](https://github.com/user-attachments/assets/89ee7182-6205-4c31-9822-d144b593a827)

#### 2.2 Attach Policies
1. Search for and attach the following policies:
   - **AmazonEC2ReadOnlyAccess** (to allow Lambda to read EC2 instance states).
   - **AmazonSNSFullAccess** (to allow Lambda to publish messages to SNS).

2. Alternatively, create a custom policy using the following JSON and attach it:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "ec2:DescribeInstances",
                "ec2:DescribeInstanceStatus"
            ],
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": "sns:Publish",
            "Resource": "<YOUR_SNS_TOPIC_ARN>"
        }
    ]
}
```

3. Click **Next**, give your role a name, e.g., `LambdaEC2StateChangeRole`.
4. Click **Create role**.
5. - -![image](https://github.com/user-attachments/assets/f9d5d091-75ae-412b-91be-f8254d7bb0df)


---

### 3. Create the Lambda Function

#### 3.1 Create a Lambda Function
1. Go to the Lambda dashboard in AWS.
2. Click **Create function**.
3. Choose **Author from scratch**.
4. Enter a name for the function, e.g., `EC2StateChangeNotifier`.
5. Select Python 3.x as the runtime.
6. Under Execution role, choose **Use an existing role** and select the role you created in Step 2.
7. Click **Create function**.
   -  ![image](https://github.com/user-attachments/assets/8f4b9c2b-afbc-44fd-9a97-a0fb6a3eacab)

#### 3.2 Add the Code
1. In the Lambda code editor, replace the default code with the following:

python

```import json
import boto3
def lambda_handler(event, context):
    try:
        # Log the received event
        print("Received event:", json.dumps(event, indent=2))
                # Extract EC2 instance details from the event
        detail = event.get("detail", {})
        instance_id = detail.get("instance-id", "Unknown")
        state = detail.get("state", "Unknown")
                # Validate the event details
        if instance_id == "Unknown" or state == "Unknown":
            print("Invalid event structure. Cannot process.")
            return {"statusCode": 400, "body": "Invalid event structure."}
                # Prepare the SNS message
        message = f"EC2 Instance {instance_id} is now {state}."
        print(f"Prepared message: {message}")
                # Send the notification via SNS
        sns_client = boto3.client('sns')
        sns_topic_arn = 'arn:aws:sns:us-east-1:225989348530:EC2StateChangeTopic'  # Replace with your SNS topic ARN
        response = sns_client.publish(
            TopicArn=sns_topic_arn,
            Message=message,
            Subject='EC2 State Change Notification'
        )
        
        # Log SNS response
        print("SNS Response:", response)
        return {"statusCode": 200, "body": "Notification sent successfully."}
    
    except Exception as e:
        print("Error:", str(e))
        raise
```

2. Replace `<YOUR_SNS_TOPIC_ARN>` with the ARN of your SNS topic (found in the SNS dashboard).
3. Click **Deploy** to save your changes.

---

### 4. Configure EventBridge to Trigger Lambda

#### 4.1 Create an EventBridge Rule
1. Navigate to **EventBridge** in the AWS Management Console.
2. Click **Rules** in the left-hand menu and then click **Create rule**.
3. Enter a name for the rule, e.g., `EC2StateChangeRule`.
4. For Event source, choose **AWS services**.
5. Under Service Name, select **EC2**.
6. For Event Type, select **EC2 Instance State-change Notification**.
- - ![image](https://github.com/user-attachments/assets/aafc6fb3-3200-402c-aa1a-8516a19ee445)
   - ![image](https://github.com/user-attachments/assets/b66b80b2-060a-430d-b380-bd991d79017a)


#### 4.2 Add Target
1. Under the Target section, choose **Lambda function**.
2. Select your Lambda function (`EC2StateChangeNotifier`).
3. Click **Create**.
- . -![image](https://github.com/user-attachments/assets/bb4f51ba-1dfe-4209-aabe-63a4920f455c)
---

### 5. Testing

#### 5.1 Start or Stop an EC2 Instance
1. Navigate to the EC2 dashboard.
2. Select an instance, then choose **Instance State > Start** or **Stop**.
3. Wait for the instance state to change.

#### 5.2 Verify SNS Notification
1. Check your email inbox.
2. You should receive a notification about the instance state change, e.g., `EC2 Instance i-0123456789abcdef is now stopped`.


- ![image](https://github.com/user-attachments/assets/5ab73c22-5395-412d-b573-eaf7ee330c0d)
- ![image](https://github.com/user-attachments/assets/155b631b-98e8-43c7-99c0-0b9069d65eb6)
- ![image](https://github.com/user-attachments/assets/af9dd713-1bf5-4a27-a73c-a000913cf052)
- ![image](https://github.com/user-attachments/assets/29ebddc2-454e-434f-a48f-96a3826a6ff4)


---

## **Step 6: Troubleshooting**

### **Common Issues and Fixes**

#### **1. Lambda Function Errors**
- **Issue**: `sns:Publish` permission error.
  - **Fix**: Ensure that the IAM role attached to the Lambda function has the `sns:Publish` permission and includes the correct SNS Topic ARN.

- **Issue**: Incorrect SNS Topic ARN.
 - **Fix**: Double-check the ARN in your Lambda code and ensure it matches the ARN of your created SNS topic.

#### **2. EventBridge Rule Not Triggering Lambda**
- **Issue**: EventBridge rule not invoking the Lambda function.
  - **Fix**: Verify that the EventBridge rule is enabled and correctly linked to your Lambda function.

#### **3. SNS Notifications Not Received**
- **Issue**: Email notifications not received.
  - **Fix**: Ensure that the email subscription is confirmed. Check your spam or junk folder for the confirmation email.

#### **4. Testing Issues**
- **Issue**: State change events not detected.
  - **Fix**: Ensure you are testing with valid state change actions like starting or stopping the instance.
  - **Fix**: Check if the instance state change events are properly logged in EventBridge.

---

## **Step 7: Cleanup (Optional)**

1. Delete the Lambda function, EventBridge rule, and SNS topic if you no longer need them.
2. Remove the IAM role if it's no longer required.

### 6. Cleanup (Optional)
1. Delete the Lambda function, EventBridge rule, and SNS topic if you no longer need them.
2. Remove the IAM role if it is no longer required.

---

This document provides step-by-step instructions to set up and test a Lambda function for monitoring EC2 instance state changes and sending notifications using SNS. Follow each step carefully to complete your assignment successfully.








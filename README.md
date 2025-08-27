# AWS EC2 Scheduler: Stop Dev Instances with Lambda & EventBridge

A serverless automation solution to reduce cloud costs by stopping EC2 instances with a specific tag on a daily schedule. This project uses an AWS Lambda function (written in Python with Boto3) triggered by an Amazon EventBridge rule to automatically stop all running instances tagged `Environment: Dev` at a specific time each day.

## Architecture Overview

1.  **Trigger:** An Amazon EventBridge schedule invokes a Lambda function based on a cron expression (e.g., every weekday at 7 PM UTC).
2.  **Logic:** The Lambda function uses the Boto3 library to:
    *   Describe all EC2 instances.
    *   Filter instances that are both `running` and have the tag `Environment: Dev`.
    *   Stop the identified instances.
3.  **Result:** Non-production EC2 instances are automatically stopped, minimizing unnecessary costs.


## Prerequisites

*   An **AWS Account** with appropriate permissions to create IAM roles, Lambda functions, EventBridge rules, and manage EC2 instances.
*   Basic familiarity with the **AWS Management Console**.
*   **Python 3.7+** and **Boto3** knowledge is helpful but not required to deploy.

## Deployment Guide

### 1. Create the IAM Execution Role

The Lambda function needs an IAM role with permissions to describe and stop EC2 instances and write logs to CloudWatch.

1.  Navigate to the **IAM Console** > **Roles** > **Create role**.
2.  Select **AWS service** as the trusted entity and choose **Lambda** as the use case. Click **Next**.
3.  Click **Create policy**. Switch to the **JSON** tab and paste the following policy:
    ```json
    {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Effect": "Allow",
                "Action": [
                    "ec2:DescribeInstances",
                    "ec2:StopInstances",
                    "logs:CreateLogGroup",
                    "logs:CreateLogStream",
                    "logs:PutLogEvents"
                ],
                "Resource": "*"
            }
        ]
    }
    ```
4.  Name the policy (e.g., `LambdaEC2SchedulerPolicy`) and click **Create policy**.
5.  Back in the role creation tab, refresh the policy list, search for your new policy, select it, and click **Next**.
6.  Name the role (e.g., `LambdaEC2SchedulerRole`), add a description, and click **Create role**.

### 2. Create the Lambda Function

1.  Navigate to the **AWS Lambda Console** > **Create function**.
2.  Choose **Author from scratch**.
3.  Set the following:
    *   **Function name:** `stop-dev-instances`
    *   **Runtime:** `Python 3.9` (or higher)
    *   **Architecture:** `x86_64`
    *   **Permissions:** Change the default execution role. Select **Use an existing role** and choose the `LambdaEC2SchedulerRole` you created.
4.  Click **Create Function**.

### 3. Deploy the Function Code

In the Lambda function's code editor, replace the default code with the following Python script:

```python
import boto3

# Initialize the EC2 client
ec2 = boto3.client('ec2')

def lambda_handler(event, context):
    # Describe all instances, checking for 'running' state and 'Environment: Dev' tag
    response = ec2.describe_instances(
        Filters=[
            {'Name': 'instance-state-name', 'Values': ['running']},
            {'Name': 'tag:Environment', 'Values': ['Dev']}
        ]
    )
    
    # Extract instance IDs from the nested response structure
    instances_to_stop = []
    for reservation in response['Reservations']:
        for instance in reservation['Instances']:
            instances_to_stop.append(instance['InstanceId'])
    
    # Stop the instances if the list is not empty
    if instances_to_stop:
        print(f"Stopping instances: {instances_to_stop}")
        ec2.stop_instances(InstanceIds=instances_to_stop)
        return {
            'statusCode': 200,
            'body': f"Successfully stopped instances: {instances_to_stop}"
        }
    else:
        print("No running Dev instances found to stop.")
        return {
            'statusCode': 200,
            'body': "No running Dev instances found to stop."
        }

## 5. Click deploy to save changes

## Test the Lambda Function Manually

1. Prerequisite: Ensure you have at least one EC2 instance running with the tag Environment: Dev.
2. In the Lambda console, click the Test tab.
3. Create a new test event. You can use the default event template as the function doesn't use the event input.
4. Name the event (e.g., ManualTest).
5. Click Test. The function should execute and return a success message, and your Dev instance(s) should begin shutting down.

5. Create the EventBridge Schedule

1. Navigate to the Amazon EventBridge Console > Schedules > Create schedule.
2. Schedule details:
   · Name: stop-dev-daily
   · Schedule group: default
   · Schedule pattern: Recurring schedule
   · Recurrence type: Cron-based (e.g., 0 23 ? * MON-FRI * to run at 11:00 PM UTC every weekday).
3. Target details:
   · Target type: AWS service
   · Select a target: Lambda function
   · Function: stop-dev-instances
4. Click Create schedule.

## 6. Verification and Testing

After deploying the entire solution, verify it works correctly.

### **Method 1: Manual Trigger Test (Recommended for Initial Setup)**
1.  Ensure you have at least one EC2 instance running with the tag `Environment: Dev`.
2.  In the Lambda console, select your `stop-dev-instances` function.
3.  Click the **Test** tab.
4.  Choose your existing `ManualTest` event.
5.  Click **Test**. The function should execute.
6.  **Check Execution Result:**
    *   The **Execution result** should show status `SUCCEEDED`.
    *   The function's output should list the stopped instances.
7.  **Check EC2 Console:** Navigate to the EC2 Instances dashboard. The targeted instances should now be in the `stopping` or `stopped` state.

### **Method 2: Testing the Scheduled Execution**
You don't have to wait until 7 PM to test the schedule.
1.  Go to the **EventBridge Console** and select your `stop-dev-daily` schedule.
2.  Click **Actions** and select **Edit**.
3.  Navigate to the **Schedule pattern** section.
4.  Change the pattern to a **One-time schedule**.
5.  Set the **Date and time** to a time 2-3 minutes in the future.
6.  Save the schedule. EventBridge will invoke your Lambda function at the specified time.
7.  Monitor the EC2 console and the Lambda function's **CloudWatch logs** to confirm it worked.

### **Method 3: Reviewing CloudWatch Logs**
The most reliable way to verify past executions is through logs.
1.  In the Lambda function console, go to the **Monitor** tab.
2.  Click **View CloudWatch logs**.
3.  In the CloudWatch window, select the most recent **log stream**.
4.  The log events will show a detailed output of the function's execution, including the list of instances it found and stopped. The timestamp of the log should match the time the schedule was triggered.

## 7. Cleaning Up

To avoid ongoing AWS charges, delete the resources you created once you are finished with this project.

1.  **Delete the EventBridge Schedule:**
    *   Navigate to the **EventBridge Schedules** console.
    *   Select the `stop-dev-daily` schedule.
    *   Click **Actions** and select **Delete**.
    *   Confirm the deletion.

2.  **Delete the Lambda Function:**
    *   Navigate to the **AWS Lambda** console.
    *   Select the `stop-dev-instances` function.
    *   Click **Actions** -> **Delete**.
    *   Confirm the deletion.

3.  **Delete the IAM Resources:**
    *   Navigate to the **IAM Console** -> **Policies**.
    *   Search for and select the `LambdaEC2SchedulerPolicy` you created.
    *   Click **Policy actions** -> **Delete**.
    *   Navigate to **IAM Console** -> **Roles**.
    *   Search for and select the `LambdaEC2SchedulerRole`.
    *   Click **Delete role** and confirm.

**⚠️ Important:** Ensure you also terminate any EC2 instances you launched for testing if you no longer need them.

# aws-lambda-ec2-scheduler
A serverless automation solution to reduce cloud costs by stopping EC2 instances with a specific tag on a daily schedule. This project uses an AWS Lambda function (written in Python with Boto3) triggered by an Amazon EventBridge rule to automatically stop all running instances tagged `Environment: Dev` at a specific time each day.

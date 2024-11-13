Serverless Architecture with AWS Lambda, RDS, and SNS

Overview
This project implements a serverless architecture using AWS Lambda for compute, Amazon RDS for database operations, and Amazon SNS for notifications. The architecture provides a scalable, maintenance-free backend that automatically handles varying workloads while maintaining cost efficiency.
Architecture Components

Core Services

AWS Lambda: Serverless compute service
Amazon RDS: Managed relational database service
Amazon SNS: Managed pub/sub messaging service

Additional Services

API Gateway: RESTful API endpoint management
IAM: Identity and access management
CloudWatch: Monitoring and logging

System Architecture
Copy┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│  API Gateway │ ─── │  AWS Lambda  │ ─── │ Amazon RDS   │
└──────────────┘     └──────────────┘     └──────────────┘
                           │
                           │
                    ┌──────────────┐
                    │ Amazon SNS   │
                    └──────────────┘
Features

Scalable Computing: Automatic scaling with AWS Lambda
Managed Database: Automated backups and maintenance with RDS
Real-time Notifications: Event-driven architecture using SNS
Cost-Effective: Pay-per-use pricing model
High Availability: Multi-AZ deployment options
Security: IAM roles and VPC configuration

Prerequisites

AWS Account with appropriate permissions
AWS CLI installed and configured
Python 3.8 or later (if using Python Lambda functions)

For this project I uploaded  zip files for the python codes,
Task 1: Downloading the source code

The code for generating the report is already written, packaged, and ready for you to deploy to AWS Lambda. 

Download the following two files to your local machine:

SalesAnalysisReportDataExtraction.py
```
import boto3
import package.pymysql as pymysql
import sys

def lambda_handler(event, context):

    # Retrieve the database connection information from the event input parameter.

    dbUrl = event['dbUrl']
    dbName = event['dbName']
    dbUser = event['dbUser']
    dbPassword = event['dbPassword']

    # Establish a connection to the Mop & Pop database, and set the cursor to return results as a Python dictionary.
    
    try:
        conn = pymysql.connect(dbUrl, user=dbUser, passwd=dbPassword, db=dbName, cursorclass=pymysql.cursors.DictCursor)
        
    except pymysql.Error as e:
        print('ERROR: Failed to connect to the Mom & Pop database.')
        print('Error Details: %d %s' % (e.args[0], e.args[1]))
        sys.exit()
    
    # Execute the query to generate the daily sales analysis result set.
    
    with conn.cursor() as cur:
        cur.execute("SELECT  c.product_group_number, c.product_group_name, a.product_id, b.product_name, CAST(sum(a.quantity) AS int) as quantity FROM order_item a, product b, product_group c WHERE b.id = a.product_id AND c.product_group_number = b.product_group GROUP BY c.product_group_number, a.product_id")
        result = cur.fetchall()

    # Close the connection.
    
    conn.close()

    # Return the result set.
    
    return {'statusCode': 200, 'body': result}

```

and 

SalesAnalysisReport
```
import boto3
import os
import json
import io
import datetime

def setTabsFor(productName):
    
    # Determine the required number of tabs between Item Name and Quantity based on the item name's length.
    
    nameLength = len(productName)
    
    if nameLength < 20:
        tabs='\t\t\t'
    elif 20 <= nameLength <= 37:
        tabs = '\t\t'
    else:
        tabs = '\t'
    
    return tabs

def lambda_handler(event, context):
    
    # Retrieve the topic ARN and the region where the lambda function is running from the environment variables.

    TOPIC_ARN = os.environ['topicARN']
    FUNCTION_REGION = os.environ['AWS_REGION']

    # Extract the topic region from the topic ARN.
    
    arnParts = TOPIC_ARN.split(':')
    TOPIC_REGION = arnParts[3]

    # Get the database connection information from the Systems Manager Parameter Store.
    
    # Create an SSM client.
    
    ssmClient = boto3.client('ssm', region_name=FUNCTION_REGION)

    # Retrieve the database URL and credentials.
    
    parm = ssmClient.get_parameter(Name='/cafe/dbUrl')
    dbUrl = parm['Parameter']['Value']

    parm = ssmClient.get_parameter(Name='/cafe/dbName')
    dbName = parm['Parameter']['Value']

    parm = ssmClient.get_parameter(Name='/cafe/dbUser')
    dbUser = parm['Parameter']['Value']

    parm = ssmClient.get_parameter(Name='/cafe/dbPassword')
    dbPassword = parm['Parameter']['Value']

    # Create a lambda client and invoke the lambda function to extract the daily sales analysis report data from the database.

    lambdaClient = boto3.client('lambda', region_name=FUNCTION_REGION)
    
    dbParameters = {"dbUrl": dbUrl, "dbName": dbName, "dbUser": dbUser, "dbPassword": dbPassword}
    response = lambdaClient.invoke(FunctionName = 'salesAnalysisReportDataExtractor', InvocationType = 'RequestResponse', Payload = json.dumps(dbParameters))

    # Convert the response payload from bytes to string, then to a Python dictionary in order to retrieve the data in the body.
    
    reportDataBytes = response['Payload'].read()
    reportDataString = str(reportDataBytes, encoding='utf-8')
    reportData = json.loads(reportDataString)
    if "body" in reportData:
        reportDataBody = reportData["body"]
    else:
        print(reportData)
        raise Exception('No body in returned data. Check the error in the cloudwatch logs.')

    # Create an SNS client, and format and publish a message containing the sales analysis report based on the extracted report data.

    snsClient = boto3.client('sns', region_name=TOPIC_REGION)
    
    # Create the message.

    # Write the report header first.
    
    message = io.StringIO()
    message.write('Sales Analysis Report'.center(80))
    message.write('\n')

    today = 'Date: ' + str(datetime.datetime.now().strftime('%Y-%m-%d'))
    message.write(today.center(80))
    message.write('\n')

    if (len(reportDataBody) > 0):

        previousProductGroupNumber = -1
        
        # Format and write a line for each item row in the report data.
        
        for productRow in reportDataBody:
            
            # Check for a product group break.
            
            if productRow['product_group_number'] != previousProductGroupNumber:
                
               # Write the product group header.
               
                message.write('\n')
                message.write('Product Group: ' + productRow['product_group_name'])
                message.write('\n\n')
                message.write('Item Name'.center(40) + '\t\t\t' + 'Quantity' + '\n')
                message.write('*********'.center(40) + '\t\t\t' + '********' + '\n')
                
                previousProductGroupNumber = productRow['product_group_number']
        
            # Write the item line.
            
            productName = productRow['product_name']
            tabs = setTabsFor(productName)
                
            itemName = productName.center(40)
            quantity = str(productRow['quantity']).center(5)
            message.write(itemName + tabs + quantity + '\n')

    else:
        
        # Write a message to indicate that there is no report data.
        
        message.write('\n')
        message.write('There were no orders today.'.center(80))

    # Publish the message to the topic.
    
    response = snsClient.publish(
        TopicArn = TOPIC_ARN,
        Subject = 'Daily Sales Analysis Report',
        Message = message.getvalue()    
    )

    # Return a successful function execution message.
    
    return {
        'statusCode': 200,
        'body': json.dumps('Sale Analysis Report sent.')
    }

```
I will create the DataExtractor Lambda function that extracts the café's sales data from an Amazon RDS database. So the Lambda function can access the RDS database instance, you must update the database security group with a rule to allow connections from the Lambda function. 
To enable this communication, you will create a security group for the Lambda function and add it as an inbound rule to the security group of the RDS instance.

    Create a security group for the Lambda function with the following settings:
        Security group name: LambdaSG
        VPC: Lab VPC
        Outbound Rules: All traffic to all addresses


    Update the DatabaseSG security group.
        Add a second inbound rule. For the new rule, configure the Type as MYSQL/Aurora. Then, in the search box to the right of Custom, type sg- and choose your new Lambda function security group as the source. Finally, choose Save rules. 

    Create a Lambda function with the following settings: 

        Function name: salesAnalysisReportDataExtractor

        Runtime: Python 3.8

        Role:  salesAnalysisReportDERole

        VPC:
            VPC: Lab VPC
            Subnets: Private subnet 1 and Private subnet 2
            Security Group: The Lambda function security group that you created

        Tip: It will take several minutes for the function to be created.

    Configure the DataExtractor Lambda function as follows:
        Code: Upload the salesAnalysisReportDataExtractor.zip file
        Description: Lambda function to extract data from database
        Handler: salesAnalysisReportDataExtractor.lambda_handler
        Memory Size: 128 MB
        Timeout (seconds): 30

   












Environment Variables
Create a .env file with the following variables:
envCopyDB_HOST=your-rds-endpoint
DB_NAME=your-database-name
DB_USER=your-database-user
DB_PASSWORD=your-database-password
SNS_TOPIC_ARN=your-sns-topic-arn
AWS_REGION=your-aws-region
Lambda Functions
User Service
pythonCopydef handler(event, context):
    # Lambda function implementation
    pass
Notification Service
pythonCopydef handler(event, context):
    # SNS notification implementation
    pass
Database Schema
sqlCopyCREATE TABLE users (
    id SERIAL PRIMARY KEY,
    username VARCHAR(50) UNIQUE NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
API Endpoints
MethodEndpointDescriptionGET/usersList usersPOST/usersCreate userPUT/users/{id}Update userDELETE/users/{id}Delete user
Monitoring and Logging

CloudWatch Metrics for Lambda functions
RDS Performance Insights
SNS delivery status tracking
Custom CloudWatch Dashboards

Security Considerations

VPC Configuration
IAM Roles and Policies
SSL/TLS Encryption
Database Security Groups
API Gateway Authentication

Cost Optimization

Lambda Concurrency Settings
RDS Instance Sizing
SNS Message Filtering
CloudWatch Log Retention

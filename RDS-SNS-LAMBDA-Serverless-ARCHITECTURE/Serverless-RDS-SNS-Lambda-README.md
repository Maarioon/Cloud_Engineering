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

![Screenshot 2024-11-06 071024](https://github.com/user-attachments/assets/6b48670c-f904-4bc4-a98c-9bbdb9c1fafd)
![Screenshot 2024-11-06 071106](https://github.com/user-attachments/assets/b43895ed-75b2-42d9-89ca-1dcea00c1fd6)
![Screenshot 2024-11-06 071430 - Copy](https://github.com/user-attachments/assets/c88ea342-862d-4418-a04d-bb5d724b9479)
![Screenshot 2024-11-06 071423](https://github.com/user-attachments/assets/ff3e0a44-ba60-445c-9386-430865aa8a4e)
![Screenshot 2024-11-06 071400](https://github.com/user-attachments/assets/6bc8f04d-4e27-4dfb-bf20-baedffd7da63)
   
![Screenshot 2024-11-06 071639 - Copy](https://github.com/user-attachments/assets/ea62b9cf-eebb-4416-846f-4a2cd374b7d2)



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


![Screenshot 2024-11-06 072544 - Copy](https://github.com/user-attachments/assets/b88ef9c1-a909-4a61-9ec9-f3399d280b42)
![Screenshot 2024-11-06 072424](https://github.com/user-attachments/assets/0a598078-f511-4b3d-9110-363cc5bbabe0)
![Screenshot 2024-11-06 072220](https://github.com/user-attachments/assets/a486116d-fa3a-4425-aff1-574349f7750a)
![Screenshot 2024-11-06 072158 - Copy](https://github.com/user-attachments/assets/7fe721ee-cf25-4b1c-87ac-0c8b7b0df26f)
![Screenshot 2024-11-06 072104](https://github.com/user-attachments/assets/3df5cede-9dff-4595-85d9-1920eb8b49a4)
![Screenshot 2024-11-06 072023](https://github.com/user-attachments/assets/d8e559a4-9b32-4b9a-93c7-bb1d21a455fd)
![Screenshot 2024-11-06 071907](https://github.com/user-attachments/assets/103cfc93-6359-41eb-9bdd-1e87b1768f1c)
![Screenshot 2024-11-06 071810](https://github.com/user-attachments/assets/15c7a79f-e20f-43cd-8bd6-b6b15ccc868f)
![Screenshot 2024-11-06 072544](https://github.com/user-attachments/assets/2a3e9b2d-2c00-423e-963e-fac283f61533)


![Screenshot 2024-11-06 072424](https://github.com/user-attachments/assets/efddd700-a197-427c-a6e2-b1f871004319)


 create the Lambda function that generates and sends the daily sales analysis report.

    Create a second Lambda function with the following settings:
        Function name: salesAnalysisReport
        Runtime: Python 3.8
        Role: salesAnalysisReportRole

    Configure the salesAnalysisReport Lambda function as follows:    
        Code: Upload the salesAnalysisReport.zip file
        Description: Lambda function to generate and send the daily sales report
        Handler: salesAnalysisReport.lambda_handler
        Memory Size: 128 MB
        Timeout (seconds): 30

![Screenshot 2024-11-06 080405](https://github.com/user-attachments/assets/c6a1a1b6-15d2-4e76-99f6-227c9b222a29)
![Screenshot 2024-11-06 075740](https://github.com/user-attachments/assets/c1d56620-11cb-4288-b8a6-4598de7bb91a)
![Screenshot 2024-11-06 075440](https://github.com/user-attachments/assets/7b6623f6-a2e6-4132-bb04-af661825f059)
![Screenshot 2024-11-06 075350](https://github.com/user-attachments/assets/e22cda28-a229-474a-a4f5-da31189d2de9)
![Screenshot 2024-11-06 075350 - Copy](https://github.com/user-attachments/assets/3c180109-265b-4960-88c6-624232ed88b8)
![Screenshot 2024-11-06 075240](https://github.com/user-attachments/assets/73a1acab-06c3-4cb3-a99b-f48c9688736d)
![Screenshot 2024-11-06 080801](https://github.com/user-attachments/assets/c5b61000-f85b-4a98-b7b8-c5b92e4f0c66)

Task 4: Creating an SNS topic

The sales analysis report uses an SNS topic to send the report to email subscribers. In this task, you will create an SNS topic and update the environment variables of the salesAnalysisReport Lambda function to store the topic Amazon Resource Name (ARN).

    Create a standard SNS topic with the following configuration:
        Name: SalesReportTopic
        Display Name: Sales Report Topic

    Update the salesAnalysisReport Lambda function by adding the following environment variable:
        Variable Name: topicARN
        Variable Value: The ARN of the topic you just created

  
![Screenshot 2024-11-06 083222](https://github.com/user-attachments/assets/4cd15556-73a6-47c3-afab-1944601e9629)
![Screenshot 2024-11-06 082226](https://github.com/user-attachments/assets/1c74f1ce-5820-4795-bea1-a13e7632826d)
![Screenshot 2024-11-06 082033](https://github.com/user-attachments/assets/ebda970f-f842-4f49-bfb9-afe8e3265742)
![Screenshot 2024-11-06 081752](https://github.com/user-attachments/assets/a8601bde-9f10-45ec-b729-aaacc29d1f2e)
![Screenshot 2024-11-06 083234](https://github.com/user-attachments/assets/b2a3f26a-c2aa-40dc-9ce1-7aeaeeab4e3d)

Task 5: Creating an email subscription to the SNS topic

To receive the sales report through email, you must create an email subscription to the topic that you created in the previous task.

    Create a new email subscription to the topic. Use an email address that you can easily access for this lab.

    Confirm the email subscription from your email client. 
    Note: If you don't receive an email confirmation, check your Junk or Spam folder.


![Screenshot 2024-11-06 083856](https://github.com/user-attachments/assets/3ae2df27-83b5-473d-b83d-cbd392c38982)
![Screenshot 2024-11-06 083811](https://github.com/user-attachments/assets/4ac781a0-b4d3-41c1-b889-02e5f5b281d7)
![Screenshot 2024-11-06 083735](https://github.com/user-attachments/assets/27c11b6c-2ba7-4dc1-8a9b-e74c64af372d)
![Screenshot 2024-11-06 083910](https://github.com/user-attachments/assets/3ad67a3c-f534-4777-930c-ee1de8330a15)

Before creating the daily reporting event, you must test that the salesAnalysisReport Lambda function works correctly.

    Create a test for the salesAnalysisReport Lambda function. 

        Tip: You don't need to worry about parameters, so enter an event name and accept the default hello-world test event.

    Run the salesAnalysisReport test. If the test succeeds, you should have an email report in a couple of minutes.

    If the Lambda function test execution failed, use the logs to review any errors, address them, and run the test again. Here are some troubleshooting tips that you can try:

        Review the logs from Amazon CloudWatch Logs for both Lambda functions:
            If you see an error about connecting to the café database, check that your security groups are configured correctly.
            If you see an error about timeout, check that the timeout is set to 30 seconds.
            If you see an error about lambda_function not found, check that you have configured the correct handler.

        Review your work to make sure that you completed all the steps.
![Screenshot 2024-11-06 084959](https://github.com/user-attachments/assets/8d50f6e3-77a5-45d3-8b7a-5b1d017bb03a)
![Screenshot 2024-11-06 084949](https://github.com/user-attachments/assets/5dd7dfaa-5b0c-4b84-9469-be850402ff59)
![Screenshot 2024-11-06 085059](https://github.com/user-attachments/assets/a6a1ae14-316a-46c5-a950-3d58d9d0179c)

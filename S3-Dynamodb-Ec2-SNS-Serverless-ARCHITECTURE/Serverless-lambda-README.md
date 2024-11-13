AWS Multi-Service Architecture
S3 • DynamoDB • EC2 • SNS • Serverless
Show Image
🌟 Architecture Overview
This project implements a scalable AWS architecture combining serverless and traditional services:

Storage: Amazon S3 for object storage
Database: DynamoDB for NoSQL data
Compute: EC2 for traditional workloads & Lambda for serverless functions
Messaging: SNS for notifications and event handling

System Architecture Diagram
Copy┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│  Amazon S3   │ ──► │  AWS Lambda  │ ──► │   DynamoDB   │
└──────────────┘     └──────────────┘     └──────────────┘
       ▲                    │                     ▲
       │                    ▼                     │
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│  EC2 Server  │ ◄─► │ Amazon SNS   │ ◄─► │  API Gateway │
└──────────────┘     └──────────────┘     └──────────────┘
🚀 Features
Storage (S3)

File upload/download capabilities
Versioning enabled
Lifecycle policies
Event-driven processing
Static website hosting (if required)

Database (DynamoDB)

Serverless NoSQL database
Auto-scaling capabilities
Point-in-time recovery
Global tables support
DAX caching (optional)

Compute (EC2)

Load-balanced instances
Auto-scaling groups
Custom AMI configuration
Security groups & NACLs
EBS volumes for persistence

Messaging (SNS)

Push notifications
Email notifications
Lambda function triggers
SQS queue integration
Mobile push notifications

🛠 Prerequisites

AWS Account with administrator access
AWS CLI installed and configured
Node.js 16.x or later
Python 3.8+ (for Lambda functions)
Terraform (optional, for IaC)

📦 Installation & Setup

 create a Lambda function that will process an inventory file. The Lambda function will read the file and insert information into a DynamoDB table.
 ![Screenshot 2024-11-13 002052](https://github.com/user-attachments/assets/3e0a13dd-6042-4fe3-8234-656ca2e8d37f)

     In the AWS Management Console, on the Services menu, choose Lambda.

    Choose Create function

    Blueprints are code templates for writing Lambda functions. Blueprints are provided for standard Lambda triggers, such as creating Amazon Alexa skills and processing Amazon Kinesis Data Firehose streams. This lab provides you with a pre-written Lambda function, so you will use the Author from scratch option.

    Configure the following settings:
        Function name: Load-Inventory
        Runtime: Python 3.9
        Expand Choose or create an execution role.
        Execution role: Use an existing role
        Existing role: Lambda-Load-Inventory-Role

    This role gives the Lambda function permissions so that it can access Amazon S3 and DynamoDB.
![Screenshot 2024-11-08 042712](https://github.com/user-attachments/assets/a4e83d17-76ea-4145-9d83-784f20c1cdb0)
![Screenshot 2024-11-08 042219](https://github.com/user-attachments/assets/d89f9825-3b1f-4ff7-9ee2-71ea9ecc4a93)
![Screenshot 2024-11-08 041658](https://github.com/user-attachments/assets/6a2151da-1bd1-4a15-857e-6925ca278bf3)
    
    Choose Create function

    Scroll down to the Code source section, and in the Environment pane, choose lambda_function.py.

    In the code editor, delete all the code.

    In the Code source editor, copy and paste the following code:
```
    # Load-Inventory Lambda function

    #

    # This function is triggered by an object being created in an Amazon S3 bucket.

    # The file is downloaded and each line is inserted into a DynamoDB table.

    import json, urllib, boto3, csv

    # Connect to S3 and DynamoDB

    s3 = boto3.resource('s3')

    dynamodb = boto3.resource('dynamodb')

    # Connect to the DynamoDB tables

    inventoryTable = dynamodb.Table('Inventory');

    # This handler is run every time the Lambda function is triggered

    def lambda_handler(event, context):

      # Show the incoming event in the debug log

      print("Event received by Lambda function: " + json.dumps(event, indent=2))

      # Get the bucket and object key from the Event

      bucket = event['Records'][0]['s3']['bucket']['name']

      key = urllib.parse.unquote_plus(event['Records'][0]['s3']['object']['key'])

      localFilename = '/tmp/inventory.txt'

      # Download the file from S3 to the local filesystem

      try:

        s3.meta.client.download_file(bucket, key, localFilename)

      except Exception as e:

        print(e)

        print('Error getting object {} from bucket {}. Make sure they exist and your bucket is in the same region as this function.'.format(key, bucket))

        raise e

      # Read the Inventory CSV file

      with open(localFilename) as csvfile:

        reader = csv.DictReader(csvfile, delimiter=',')

        # Read each row in the file

        rowCount = 0

        for row in reader:

          rowCount += 1

          # Show the row in the debug log

          print(row['store'], row['item'], row['count'])

          try:

            # Insert Store, Item and Count into the Inventory table

            inventoryTable.put_item(

              Item={

                'Store':  row['store'],

                'Item':   row['item'],

                'Count':  int(row['count'])})

          except Exception as e:

             print(e)

             print("Unable to insert data into DynamoDB table".format(e))

        # Finished!

        return "%d counts inserted" % rowCount
```
    Examine the code. It performs the following steps:
        Download the file from Amazon S3 that triggered the event
        Loop through each line in the file
        Insert the data into the DynamoDB Inventory table

    Choose Deploy to save your changes.
![Screenshot 2024-11-08 041013](https://github.com/user-attachments/assets/621b7c01-2ef3-4877-8a1f-0e507ba98200)
![Screenshot 2024-11-08 040820](https://github.com/user-attachments/assets/9bf519fe-7758-42f6-b195-bcffc2e59a60)
![Screenshot 2024-11-08 042916](https://github.com/user-attachments/assets/46f8292a-5b08-4d96-af1f-b7a48adf303f)
![Screenshot 2024-11-08 042901](https://github.com/user-attachments/assets/d3429fa3-510a-45f6-92b7-1d151120b0a6)
![Screenshot 2024-11-08 042916](https://github.com/user-attachments/assets/c6779d1b-f1ed-4fe8-8abc-fa2c96cbc478)


    Next, you will configure Amazon S3 to trigger the Lambda function when a file is uploaded.

Configuring an Amazon S3 event

Stores from around the world provide inventory files to load into the inventory tracking system. Instead of uploading their files via FTP, the stores can upload them directly to Amazon S3. They can upload the files through a webpage, a script, or as part of a program. 
When a file is received, it triggers the Lambda function. This Lambda function will then load the inventory into a DynamoDB table.

create an S3 bucket and configure it to trigger the Lambda function.

    On the Services menu, choose S3.

    Choose Create bucket

    Each bucket must have a unique name, so you will add a random number to the bucket name. For example: inventory-123

    For Bucket name enter: inventory-<number> (Replace with a random number)

    Choose Create bucket

    You might receive an error that states: The requested bucket name is not available. If you get this error, choose the first Edit link, change the bucket name, and try again until the bucket name is accepted.

    You will now configure the bucket to automatically trigger the Lambda function when a file is uploaded.

    Choose the name of your inventory- bucket.

    Choose the Properties tab.

    Scroll down to Event notifications.

    You will configure an event to trigger when an object is created in the S3 bucket.

    Click Create event notification then configure these settings:
     
        Name: Load-Inventory
        Event types: All object create events
        Destination: Lambda Function
        Lambda function: Load-Inventory
        Choose Save changes
   
    When an object is created in the bucket, this configuration tells Amazon S3 to trigger the Load-Inventory Lambda function that you created earlier.

    Your bucket is now ready to receive inventor

![Screenshot 2024-11-08 043415](https://github.com/user-attachments/assets/f28ce999-bcd6-4827-93ef-35578c29ec90)
![Screenshot 2024-11-08 043407](https://github.com/user-attachments/assets/304489e7-189c-4d65-a67a-741e974e5d87)
![Screenshot 2024-11-08 043245](https://github.com/user-attachments/assets/1c1fe541-21b7-421a-841b-7c8d4e45f77c)
![Screenshot 2024-11-08 043118](https://github.com/user-attachments/assets/07f0a94e-c23c-4443-9053-6c7fe1a7cc0a)
![Screenshot 2024-11-08 043930](https://github.com/user-attachments/assets/c96453e6-8085-4fa0-8435-04d91f8ef77c)
![Screenshot 2024-11-08 043815](https://github.com/user-attachments/assets/934275fc-d6be-4469-82e4-3d4854dd02f3)
![Screenshot 2024-11-08 043441](https://github.com/user-attachments/assets/dcc89359-d274-48b9-9edd-a281664d97d8)

Use any CSV file of your choice and upload it
create an S3 bucket and configure it to trigger the Lambda function.

    On the Services menu, choose S3.

    Choose Create bucket

    Each bucket must have a unique name, so you will add a random number to the bucket name. For example: inventory-123

    For Bucket name enter: inventory-<number> (Replace with a random number)

    Choose Create bucket

    You might receive an error that states: The requested bucket name is not available. If you get this error, choose the first Edit link, change the bucket name, and try again until the bucket name is accepted.

    You will now configure the bucket to automatically trigger the Lambda function when a file is uploaded.

    Choose the name of your inventory- bucket.

    Choose the Properties tab.

    Scroll down to Event notifications.

    You will configure an event to trigger when an object is created in the S3 bucket.

    Click Create event notification then configure these settings:
        Name: Load-Inventory
        Event types: All object create events
        Destination: Lambda Function
        Lambda function: Load-Inventory
        Choose Save changes

    When an object is created in the bucket, this configuration tells Amazon S3 to trigger the Load-Inventory Lambda function that you created earlier.

    Your bucket is now ready to receive inventor
These files are the inventory files that you can use to test the system. They are comma-separated values (CSV) files. The following example shows the contents of the Berlin file:

  store,item,count

  Berlin,Echo Dot,12

  Berlin,Echo (2nd Gen),19

  Berlin,Echo Show,18

  Berlin,Echo Plus,0

  Berlin,Echo Look,10

  Berlin,Amazon Tap,15

    In the console, return to your S3 bucket by choosing the Objects tab.

    Choose Upload

    Choose Add files, and select one of the inventory CSV files. (You can choose any inventory file.)

    Choose Upload

    Amazon S3 will automatically trigger the Lambda function, which will load the data into a DynamoDB table.

    A serverless Dashboard application has been provided for you to view the results.

    At the top of these instructions, choose the Details button, and to the right of AWS, choose the Show button.

    From the Credentials window, copy the Dashboard URL.

    Open a new web browser tab, paste the URL, and press ENTER.

    The dashboard application will open and display the inventory data that you loaded into the bucket.
The data is retrieved from DynamoDB, which proves that the upload successfully triggered the Lambda function.

![Screenshot 2024-11-08 043930](https://github.com/user-attachments/assets/ef870d4c-b649-4a41-a5a9-e49ce9f9b5e8)
![Screenshot 2024-11-08 043815](https://github.com/user-attachments/assets/d6d8cf62-61f6-48e8-b6ff-d4e2c8460993)
![Screenshot 2024-11-08 044242](https://github.com/user-attachments/assets/42197a92-1a8a-4007-a183-740bff9032bc)
![Screenshot 2024-11-08 044138](https://github.com/user-attachments/assets/4612b23c-836d-4bce-b36d-77f42df6673c)
![Screenshot 2024-11-08 044127](https://github.com/user-attachments/assets/dc5ccb9f-fe3e-48c2-b3ed-9b24371a9e64)

 If the dashboard application does not display any information, ask your instructor to help you diagnose the problem.

The dashboard application is served as a static webpage from Amazon S3. The dashboard authenticates via Amazon Cognito as an anonymous user, which provides sufficient permissions for the dashboard to retrieve data from DynamoDB.

You can also view the data directly in the DynamoDB table.

    On the Services menu, choose DynamoDB.

    In the left navigation pane, choose Tables.

    Choose the Inventory table.

    Choose the Items tab.

    The data from the inventory file will be displayed. It shows the store, item and inventory count.


Task 4: Configuring notifications

You want to notify inventory management staff when a store runs out of stock for an item. For this serverless notification functionality, you will use Amazon SNS.
Amazon SNS is a flexible, fully managed publish/subscribe messaging and mobile notifications service. It delivers messages to subscribing endpoints and clients. With Amazon SNS, you can fan out messages to a large number of subscribers, including distributed systems and services, and mobile devices.

    On the Services menu, choose Simple Notification Service.

    In the Create topic box, for Topic name, enter: NoStock. Keep Standard selected.

    Choose Create topic

    To receive notifications, you must subscribe to the topic. You can choose to receive notifications via several methods, such as SMS and email.

    In the lower half of the page, choose Create subscription and configure these settings:
        Protocol: Email
        Endpoint: Enter your email address
        Choose Create subscription

    After you create an email subscription, you will receive a confirmation email message. Open the message and choose the Confirm subscription link.

    Any message that is sent to the SNS topic will be forwarded to your email.
![Screenshot 2024-11-06 084949](https://github.com/user-attachments/assets/b8b56344-88d7-439b-8fa3-60728b42162f)
![Screenshot 2024-11-06 083735](https://github.com/user-attachments/assets/77a7b88b-8bd8-4daf-84e6-193914bb124a)

![Screenshot 2024-11-06 083811](https://github.com/user-attachments/assets/03eeec0f-a692-41dd-bc8b-164622c5149b)
![Screenshot 2024-11-06 083856](https://github.com/user-attachments/assets/efae19d0-a4c9-4036-b54e-444512dcb050)


Hybrid Data Migration using AWS Storage Gateway
Overview
Migrating data from on-premises storage to the cloud can be a complex and time-consuming process. This project demonstrates how to leverage AWS Storage Gateway to simplify the hybrid data migration process and seamlessly integrate on-premises storage with the AWS cloud.
AWS Storage Gateway Overview
AWS Storage Gateway is a hybrid cloud storage service that provides on-premises access to virtually unlimited cloud storage. It offers three gateway types:

File Gateway: Provides network file system (NFS) and Server Message Block (SMB) access to data stored in Amazon S3.
Volume Gateway: Provides iSCSI-based block storage, backed by Amazon S3 and Amazon Elastic Block Store (EBS).
Tape Gateway: Provides virtual tape library (VTL) access, backed by Amazon S3 and Amazon Glacier.

For this project, we'll be focusing on the File Gateway to enable hybrid data migration.
Migration Architecture
The hybrid data migration process using AWS Storage Gateway involves the following steps:

Deploy AWS Storage Gateway: Set up a file gateway on-premises or in the cloud, connecting it to your existing storage infrastructure.
Configure Data Synchronization: Establish synchronization between the on-premises storage and the Amazon S3 bucket, allowing data to be replicated to the cloud.
Migrate Data to AWS: Use the file gateway to seamlessly migrate data from your on-premises storage to the Amazon S3 bucket.
Optimize and Manage: Manage the hybrid storage environment, monitor data usage, and optimize the migration process as needed.

Copy┌───────────────┐     ┌───────────────┐
│  On-Premises  │     │     AWS       │
│    Storage    │     │    Cloud     │
│              │     │              │
│              │     │              │
└───────────────┘     └───────────────┘
        │                     │
        │                     │
┌───────────────┐     ┌───────────────┐
│ AWS Storage   │     │   Amazon S3  │
│    Gateway    │     │              │
│    (File)     │     │              │
└───────────────┘     └───────────────┘
Benefits of Using AWS Storage Gateway

Seamless Data Migration: The file gateway provides a familiar NFS/SMB interface, allowing you to migrate data using your existing storage management tools and processes.
Hybrid Cloud Storage: Maintain low-latency access to your frequently accessed data on-premises, while seamlessly tiering less-accessed data to Amazon S3.
Scalable Cloud Storage: Leverage the virtually unlimited storage capacity and durability of Amazon S3 to accommodate growing data needs.
Data Lifecycle Management: Automatically move data between different Amazon S3 storage classes (e.g., S3 Standard, S3 Glacier) based on your data access patterns and cost optimization requirements.
Reduced Storage Costs: Offload less-accessed data to the cost-effective Amazon S3 storage, while keeping your on-premises storage footprint minimal.
Disaster Recovery: Use the file gateway to set up a cloud-based disaster recovery solution, with data replicated to Amazon S3.

Implementation Steps

The lab environment I used were a total of three AWS Regions. A Linux EC2 instance that emulates an on-premises server is deployed to the us-east-1 (N. Virginia) Region. The Storage Gateway virtual appliance is deployed to the same Region as the Linux server. In a real-world scenario, the appliance would be deployed in a VMware vSphere or Microsoft Hyper-V environment, or as a physical Storage Gateway appliance.

The primary S3 bucket is created in the us-east-2 (Ohio) Region. Data from the Linux host is copied to the primary S3 bucket. This bucket can also be called the source.

The secondary S3 bucket is created in the us-west-2 (Oregon) Region. This secondary bucket is the target for the cross-Region replication policy. It can also be called the destination.
![Screenshot 2024-11-13 080325](https://github.com/user-attachments/assets/3bf63102-27d1-465d-863e-aff59dcaf190)

Creating the primary and secondary S3 buckets

Before you configure the File Gateway, you must create the primary S3 bucket (or the source) where you will replicate the data. You will also create the secondary bucket (or the destination) that will be used for cross-Region replication.

     In the search box to the right of  Services, search for and choose S3 to open the S3 console.
 
    Choose Create bucket then configure these settings:

        Bucket name: Create a name that you can remember easily. It must be globally unique.

        Region: US East (Ohio) us-east-2

        Bucket Versioning: Enable

         For cross-Region replication, you must enable versioning for both the source and destination buckets.  

     Choose Create bucket

    Repeat the previous steps in this task to create a second bucket with the following configuration:
        Bucket name: Create a name you can easily remember. It must be globally unique.
        Region: US West (Oregon) us-west-2
        Versioning: Enable

![Screenshot 2024-11-08 085719](https://github.com/user-attachments/assets/f12c1f04-36fd-4872-961e-41fdcb52156f)
![Screenshot 2024-11-08 085557](https://github.com/user-attachments/assets/6d8273c7-771e-4032-8a0f-afb220ceaed6)
![Screenshot 2024-11-08 085233](https://github.com/user-attachments/assets/d7e81bdf-4b0c-42a5-8cf9-4eb07f0f79b6)
![Screenshot 2024-11-08 085858](https://github.com/user-attachments/assets/baf63870-b779-4cd7-b9c3-7b3acf398991)

nabling cross-Region replication

Now that you created your two S3 buckets and enabled versioning on them, you can create a replication policy.

 

    Select the name of the source bucket that you created in the US East (Ohio) Region.

     

    Select the Management tab and under Replication rules select Create replication rule

     

    Configure the Replication rule:

        Replication rule name: crr-full-bucket

        Status Enabled

        Source bucket:
            For Choose a rule scope, select  Apply to all objects in the bucket

        Destination:

             Choose a bucket in this account

            Choose Browse S3 and select the bucket you created in the US West (Oregon) Region.

            Select Choose path

            IAM role: S3-CRR-Role 

     Note:  To find the AWS Identity and Access Management (IAM) role, in the search box, enter: S3-CRR (This role was pre-created with the required permissions for this lab)

    Choose Save. When prompted, if you want to replicate existing objects, choose No, and then choose Submit

    Note: there are no objects currently in the bucket, so the answer will have no effect in this case.

    Return to and select the link to the bucket you created in the US East (Ohio) Region.

    Choose Upload to upload a file from your local computer to the bucket.

    For this lab, use a small file that does not contain sensitive information, such as a blank text file.

    Choose Add files, locate and open the file, then choose Upload 

    Wait for the file to upload, then choose Close. Return to the bucket you created in the US West (Oregon) Region. 

    The file that you uploaded should also now have been copied to this bucket.

    Note: You may need to refresh  the console for the object to appear.

 ![Screenshot 2024-11-08 092943](https://github.com/user-attachments/assets/9d6a6894-d81c-469c-b97b-3b638c719dd1)
![Screenshot 2024-11-08 092854](https://github.com/user-attachments/assets/6b014ee7-9ca2-4c80-a676-80901f95e301)
![Screenshot 2024-11-08 091536](https://github.com/user-attachments/assets/af24451d-a335-465c-9c10-515cb83bdf4b)
![Screenshot 2024-11-08 091355](https://github.com/user-attachments/assets/a2f272d7-da0c-442f-b3c7-4f635d6e0e50)
![Screenshot 2024-11-08 093246](https://github.com/user-attachments/assets/e2c167df-6f9e-407e-9e02-d61bf55a2ede)
![Screenshot 2024-11-08 093201](https://github.com/user-attachments/assets/0c971224-d7cb-402a-ae95-67c3d9cd83a2)
![Screenshot 2024-11-08 093256](https://github.com/user-attachments/assets/e21aeb79-f62b-432a-a6f3-f39b8155d8b4)

Configuring the File Gateway and creating an NFS file share

In this task, you will deploy the File Gateway appliance as an Amazon Elastic Compute Cloud (Amazon EC2) instance. You will then configure a cache disk, select an S3 bucket to synchronize your on-premises files to, and select an IAM policy to use. Finally, you will create an NFS file share on the File Gateway.

 

    In the search box to the right of  Services, search for and choose Storage Gateway to open the Storage Gateway console.

 

    At the top-right of the console, verify that the current Region is N. Virginia.

     

    Choose Create gateway then begin configuring the Step 1: Set up gateway settings:

        Gateway name: File Gateway

        Gateway time zone: Choose GMT -5:00 Eastern Time (US & Canada), Bogota, Lima

        Gateway type: Amazon S3 File Gateway
![Screenshot 2024-11-08 093442](https://github.com/user-attachments/assets/70e1a4bc-c47c-41a1-937b-2f3f5ba96ebe)
![Screenshot 2024-11-08 093256](https://github.com/user-attachments/assets/18a137a6-6f16-4184-9751-9ec0e9dd3162)
![Screenshot 2024-11-08 093639](https://github.com/user-attachments/assets/c9443c68-3c59-45f3-81a0-5a27ea0b2224)
![Screenshot 2024-11-08 093459](https://github.com/user-attachments/assets/e0db6af0-3587-4238-a984-2b3004c0943b)

        Host platform: choose Amazon EC2. Choose Customize your settings. Then choose the Launch instance button.

        A new tab opens to the EC2 instance launch wizard. This link automatically selects the correct Amazon Machine Image (AMI) that must be used for the File Gateway appliance.

    In the Launch an instance screen, begin configuring the gateway as described:

        Name: File Gateway Appliance

        AMI from catalog: Accept the default aws-storage-gateway AMI.

        Instance type: Select the t2.xlarge instance type

        Note: t2.xlarge is the only instance type that you can select in this lab environment. If you select any other instance type, it will result in an error message when you attempt to launch the instance.

         The t2.xlarge instance type is used only as an example in this lab. For correct appliance sizing when you deploy a Storage Gateway appliance, refer to the Storage Gateway documentation.

        Key pair name - required: choose the existing vockey key pair. 

        Note: This SSH key pair is provided on the Details > Show page for this lab.
![Screenshot 2024-11-08 093720](https://github.com/user-attachments/assets/782b17fd-c5a2-4313-bac2-ed33e7d7c31e)
![Screenshot 2024-11-08 093639](https://github.com/user-attachments/assets/57877e84-b087-42e7-b923-142d5272dc24)
![Screenshot 2024-11-08 093720](https://github.com/user-attachments/assets/36ca6b9d-9e9f-4d82-9481-094b9124a3bb)
![Screenshot 2024-11-08 093654](https://github.com/user-attachments/assets/c67537ba-e594-4cfa-8253-277e36c6b3c2)
![Screenshot 2024-11-08 093647](https://github.com/user-attachments/assets/d8c6b7da-2383-45ed-b95b-579e69021f43)


Configure the network and security group settings for the gateway.

    Next to Network settings, choose Edit, then configure: 
        VPC: On-Prem-VPC
        Subnet: On-Prem-Subnet
        Auto-assign public IP: Enable
        Under Firewall (security groups), choose  Select an existing security group.

    For Common security groups: 

        Select the security group with FileGatewayAccess in the name

        Note: This security group is configured to allow traffic through ports 80 (HTTP), 443 (HTTPS), 53 (DNS), 123 (NTP), and 2049 (NFS). These ports enable the activation of the File Gateway appliance. They also enable connectivity from the Linux server to the NFS share that you will create on the File Gateway.

        For additional information about the ports used by Storage Gateway, refer to the Storage Gateway documentation.

        Also select the security group with OnPremSshAccess in the name

        Note: This security group is configured to allow Secure Shell (SSH) connections on port 22.

        Verify that both security group now appear as selected (details on each will appear in boxes in the console). 

        Tip: You may need to choose Show all selected to see them both.
![Screenshot 2024-11-08 093804](https://github.com/user-attachments/assets/5d79b92f-76d7-4c00-8621-2822ecaa2db6)
![Screenshot 2024-11-08 093920](https://github.com/user-attachments/assets/90286af6-a0cf-435b-a7a2-555e213b3df1)
![Screenshot 2024-11-08 093831](https://github.com/user-attachments/assets/e0cc8bfc-18d8-471f-96ff-ad897339fbe4)

    Configure the storage settings for the gateway.

        In the Configure storage panel, notice there is already an entry to create one 80GiB root volume.

        Choose Add new volume 

        Set the size of the EBS volume to 150GiB

    Finish creating the gateway.

        In the Summary panel on the right, keep the number of instances set to 1, and choose Launch instance

        A Success message displays.

        Choose View all instances

        Your File Gateway Appliance instance will take a few minutes to initialize.

    Monitor the status of the deployment and wait for Status Checks to complete.

    Tip: Choose the refresh  button to more quickly learn the status of the instance.

    Select your File Gateway instance, then in the Details tab below, locate the Public IPv4 address and copy it. 

    You will use this IP address when you complete the File Gateway deployment.

    Return to the AWS Storage Gateway tab in your browser. It should still be at the Set up gateway on Amazon EC2 screen.

    Check the box next to I completed all the steps above and launched the EC2 instance, then choose Next
        Configure the storage settings for the gateway.

        In the Configure storage panel, notice there is already an entry to create one 80GiB root volume.

        Choose Add new volume 

        Set the size of the EBS volume to 150GiB

         

    Finish creating the gateway.

        In the Summary panel on the right, keep the number of instances set to 1, and choose Launch instance

        A Success message displays.

        Choose View all instances

        Your File Gateway Appliance instance will take a few minutes to initialize.

    Monitor the status of the deployment and wait for Status Checks to complete.

    Tip: Choose the refresh  button to more quickly learn the status of the instance.

    Select your File Gateway instance, then in the Details tab below, locate the Public IPv4 address and copy it. 

    You will use this IP address when you complete the File Gateway deployment.

    Return to the AWS Storage Gateway tab in your browser. It should still be at the Set up gateway on Amazon EC2 screen.

    Check the box next to I completed all the steps above and launched the EC2 instance, then choose Next

![Screenshot 2024-11-08 095525](https://github.com/user-attachments/assets/5c98f8a1-ecf2-475d-87aa-4c981d05eeb5)
![Screenshot 2024-11-08 095518](https://github.com/user-attachments/assets/0873abb9-36a7-4685-a62f-b9d5351cabb4)
![Screenshot 2024-11-08 095440](https://github.com/user-attachments/assets/4f834390-4b4d-4d3b-89d0-972d2c6b4984)
![Screenshot 2024-11-08 095429](https://github.com/user-attachments/assets/ec03e035-160c-4be2-974f-cdda1a72b35e)
![Screenshot 2024-11-08 095404](https://github.com/user-attachments/assets/a8a2637f-ff09-4c27-9bae-a8462c5bf546)
![Screenshot 2024-11-08 095345](https://github.com/user-attachments/assets/c3843032-f45f-4f3e-93cb-ae1a851f850d)
![Screenshot 2024-11-08 095251](https://github.com/user-attachments/assets/ef806b4f-796d-4bc2-aa1b-afb1f6a904fe)
![Screenshot 2024-11-08 094414](https://github.com/user-attachments/assets/28ebf04b-3ab8-4eb1-821e-97916d770614)
![Screenshot 2024-11-08 094348](https://github.com/user-attachments/assets/e2add789-f4a0-4758-867b-ace7815dd54f)
![Screenshot 2024-11-08 093957](https://github.com/user-attachments/assets/eda50c87-de77-438e-a0ea-f2758e2eb872)
![Screenshot 2024-11-08 095618](https://github.com/user-attachments/assets/2722aac3-b84e-428b-a9ec-fb33c6812860)
![Screenshot 2024-11-08 095558](https://github.com/user-attachments/assets/54461c61-94b4-4aab-bbe1-dff24267bf60)

    Configure the Step 2: Connect to AWS settings:

        In the Gateway connection options:
            For IP address, paste in the IPv4 Public IP address that you copied from your File Gateway Appliance instance

        For the Service endpoint, select Publicly accessible.

        Choose Next

         

    In the Step 3: Review and activate settings screen choose Activate gateway

     

    Configure the Step 4: Configure gateway settings:

        CloudWatch log group: Deactivate logging

        CloudWatch alarms: No Alarm

        A Successfully activated gateway File Gateway Appliance message displays. 

        In the Configure cache storage panel, you will see that a message the local disks are loading.  

        Wait for the local disks status to show that it finished processing (approximately 1 minute).

        Choose Configure

 

    Start creating a file share.
        Wait for File Gateway status to change to Running.
        From the left side panel, choose File shares.
        Choose Create file share.

    On the Create file share screen, configure these settings:
        Gateway: Select the name of the File Gateway that you just created (which should be File Gateway Appliance)
        File share protocol: NFS
        Amazon S3 bucket name: Choose the name of the source bucket that you created in the US East (Ohio) us-east-2 Region in Task 1.
        Choose Customize configuration
        For File share name use share and choose Next.

    On the Amazon S3 storage settings screen, configure these settings:

        Storage class for new objects: S3 Standard

        Object metadata:
             Guess MIME type
             Gateway files acccessible to S3 bucket owner
             Enable Requester Pays

        Access your S3 bucket: Use an existing IAM role

        IAM role: Paste the FgwIamPolicyARN, which you can retrieve by following these instructions –
            Choose the Details dropdown menu above these instructions
            Select Show
            Copy the FgwIamPolicyARN value

        Choose Next
          In the File access settings screen, accept the default settings.

    Note: You might get a warning message that the file share is accessible from anywhere. For this lab, you can safely disregard this warning. In a production environment, you should always create policies that are as restrictive as possible to prevent unwanted or malicious connections to your instances.

        Choose Next

    Scroll to the bottom of the Review and create screen, then select Create 

    Monitor the status of the deployment and wait for Status to change to Available, which takes less than a minute.

    Note: You can choose the refresh  button occasionally to notice more quickly when the status has changed.

    Select the file share that you just created by choosing the link.

![Screenshot 2024-11-08 100838](https://github.com/user-attachments/assets/898ad399-cb39-429d-82e6-0e98549bc870)
![Screenshot 2024-11-08 100108](https://github.com/user-attachments/assets/ccad64d7-84d8-453e-a1ac-16b69eef92ad)
![Screenshot 2024-11-08 095845](https://github.com/user-attachments/assets/8efcd8cf-5b7c-4373-b0c8-ae9478451d54)
![Screenshot 2024-11-08 095704](https://github.com/user-attachments/assets/2d5b3cd0-75c2-451b-8b4d-82c483038a70)
![Screenshot 2024-11-08 100936](https://github.com/user-attachments/assets/831f78b9-27ba-4a11-ad37-47657d9ce424)
![Screenshot 2024-11-08 100928](https://github.com/user-attachments/assets/ab430e3d-2465-4dfa-9ed8-27a28d2ec8dc)

You can mout your File on Linux
    In the File access settings screen, accept the default settings.

    Note: You might get a warning message that the file share is accessible from anywhere. For this lab, you can safely disregard this warning. In a production environment, you should always create policies that are as restrictive as possible to prevent unwanted or malicious connections to your instances.

        Choose Next

    Scroll to the bottom of the Review and create screen, then select Create 

    Monitor the status of the deployment and wait for Status to change to Available, which takes less than a minute.

    Note: You can choose the refresh  button occasionally to notice more quickly when the status has changed.

    Select the file share that you just created by choosing the link.

    At the bottom of the screen, note the command to mount the file share on Linux. You will need it for the next task.

    Linux Mount Command
        In the File access settings screen, accept the default settings.

    Note: You might get a warning message that the file share is accessible from anywhere. For this lab, you can safely disregard this warning. In a production environment, you should always create policies that are as restrictive as possible to prevent unwanted or malicious connections to your instances.

        Choose Next

         

    Scroll to the bottom of the Review and create screen, then select Create 

    Monitor the status of the deployment and wait for Status to change to Available, which takes less than a minute.

    Note: You can choose the refresh  button occasionally to notice more quickly when the status has changed.

     

    Select the file share that you just created by choosing the link.

     

    At the bottom of the screen, note the command to mount the file share on Linux. You will need it for the next task.

    Linux Mount Command
    You can use any terminal you like I am using Putty

    When you are prompted to allow the first connection to this remote SSH server, enter yes.

    Because you are using a key pair for authentication, you are not prompted for a password.
![Screenshot 2024-11-08 101230](https://github.com/user-attachments/assets/285c040d-3695-4997-bc6d-0002591ece4d)
![Screenshot 2024-11-08 101222](https://github.com/user-attachments/assets/55d0f3bf-8311-49eb-9a03-64af3f11250a)
![Screenshot 2024-11-08 101133](https://github.com/user-attachments/assets/eb4510b9-1cbe-4fb2-aa71-9f60e5d8f55c)
![Screenshot 2024-11-08 101115](https://github.com/user-attachments/assets/6f463053-aeaa-4b31-8fa0-310d76070f73)

    On the Linux instance, to view the data that exists on this server, enter the following command:
    ```
    ls /media/data
    ```
    You should see 20 image files in the .png format.

 

    Create the directory that will be used to synchronize data with your S3 bucket by using the following command:
    ```
    sudo mkdir -p /mnt/nfs/s3
    ```

    Mount the file share on the Linux instance by using the command that you located in the Storage Gateway file shares details screen at the end of the last task.

    sudo mount -t nfs -o nolock,hard <File-Gateway-appliance-private-IP-address>:/share /mnt/nfs/s3

    Notice that the command starts with sudo and ends with /mnt/nfs/s3. 

    For example:
    ```
    sudo mount -t nfs -o nolock,hard 10.10.1.33:/share /mnt/nfs/s3
    ```
    ![Screenshot 2024-11-08 101755](https://github.com/user-attachments/assets/5e912dae-f721-4a57-99b2-ad67fbf5722f)
![Screenshot 2024-11-08 101720](https://github.com/user-attachments/assets/6300d104-d6f3-4734-b7c9-3928bdbd178e)
![Screenshot 2024-11-08 101512](https://github.com/user-attachments/assets/e57583c9-16a5-4ed7-9fd8-0a1d8fe75c98)
erify that the share was mounted correctly by entering the following command:

df -h

The output of the command should be similar to the following example:

[ec2-user@ip-10-10-1-210 ~]$ df -h

Filesystem                  Size  Used Avail Use% Mounted on

devtmpfs                    483M   64K  483M   1% /dev

tmpfs                       493M     0  493M   0% /dev/shm

/dev/xvda1                  7.8G  1.1G  6.6G  14% /

10.10.1.33:/share  8.0E     0  8.0E   0% /mnt/nfs/s3

Now that you created the mount point, you can copy the data that you want to migrate to Amazon S3 into the share by using this command:

cp -v /media/data/*.png /mnt/nfs/s3

![Screenshot 2024-11-08 101755](https://github.com/user-attachments/assets/0153b120-f2b3-4d6b-a9df-5b48c3c3c504)
![Screenshot 2024-11-08 101720](https://github.com/user-attachments/assets/2c705810-7074-4cf2-9ce5-159f2a66ec12)

Verifying that the data is migrated

You have finished configuring the gateway and copying data into the NFS share. Now, you will verify that the configuration works as intended.

    In the  Services search box, search for and choose S3 to open the S3 console.

    Select the bucket that you created in the US East (Ohio) Region.

    Verify that the 20 image files are listed.

     Note: You might need to choose the refresh  icon in the S3 console.

    Return to the Buckets page and select the bucket that you created in the US West (Oregon) Region. 

    Verify that the images files were replicated to this bucket, based on the policy that you created earlier.

     Note: S3 Object replication can take up to 15 minutes to complete. Keep refreshing until you see the replicated objects. 
![Screenshot 2024-11-08 101955](https://github.com/user-attachments/assets/ed3cd9f1-1554-4d8a-b958-25a35e5287d6)
![Screenshot 2024-11-08 101946](https://github.com/user-attachments/assets/3b4be69f-815a-4559-b763-daa69907efb1)
![Screenshot 2024-11-08 101901](https://github.com/user-attachments/assets/316862d1-3d83-49d1-a84a-ffce6452650e)
![Screenshot 2024-11-08 101836](https://github.com/user-attachments/assets/a1c1aef1-5a71-48b8-a65b-0a47be26ed96)


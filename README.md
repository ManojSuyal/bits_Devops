# Cloud

#Assignment

1.	Using AWS lambda, write a program to monitor an S3 Uri. Whenever a user uploads data into the S3 storage, the program should capture the details. At the end of the day the program must send out an email to select users displaying the following information
a.	S3 Uri
b.	Object Name
c.	Object Size
d.	Object type
In addition to the above, the program should create a thumbnail and store it in the same uri in case the user uploads an image (.jpg/jpeg/png)
"

#Step1 1
Create an S3 Bucket: If you haven't already, create an S3 bucket where the user will upload data.
Set up Lambda Function:
Go to the AWS Lambda console.
Click on "Create function" and choose "Author from scratch".
Give your function a name, choose the runtime as Python 3.8, and select an existing role or create a new one with necessary permissions to access S3 and SES.
Click on "Create function".
Write Lambda Function Code:
python
Copy code
import json
import boto3
import os
import mimetypes
from email.mime.multipart import MIMEMultipart
from email.mime.text import MIMEText
from email.mime.application import MIMEApplication
from email.mime.image import MIMEImage
from email.utils import formatdate

s3 = boto3.client('s3')
ses = boto3.client('ses')

def lambda_handler(event, context):
    bucket_name = event['Records'][0]['s3']['bucket']['name']
    object_key = event['Records'][0]['s3']['object']['key']
    object_size = event['Records'][0]['s3']['object']['size']
    object_type = mimetypes.guess_type(object_key)[0]
    
    if object_type is not None and object_type.startswith('image'):
        create_thumbnail(bucket_name, object_key)
    
    send_email(bucket_name, object_key, object_size, object_type)
    
    return {
        'statusCode': 200,
        'body': json.dumps('Email sent successfully!')
    }

def create_thumbnail(bucket_name, object_key):
    s3.download_file(bucket_name, object_key, '/tmp/image.jpg') # Change file extension if needed
    # Code to create thumbnail using PIL or any other library
    # Save thumbnail to /tmp/thumbnail.jpg
    s3.upload_file('/tmp/thumbnail.jpg', bucket_name, 'thumbnails/' + os.path.basename(object_key))

def send_email(bucket_name, object_key, object_size, object_type):
    recipients = ['user1@example.com', 'user2@example.com'] # Add email addresses of users to receive email
    sender = 'your-email@example.com' # Change this to your email address
    subject = 'S3 Upload Notification'

    body = f"""\
    <html>
      <head></head>
      <body>
        <p>Details of the uploaded object:</p>
        <ul>
          <li><strong>S3 Uri:</strong> s3://{bucket_name}</li>
          <li><strong>Object Name:</strong> {object_key}</li>
          <li><strong>Object Size:</strong> {object_size} bytes</li>
          <li><strong>Object Type:</strong> {object_type}</li>
        </ul>
        <p>Thumbnail:</p>
        <img src="cid:thumbnail">
      </body>
    </html>
    """

    msg = MIMEMultipart()
    msg['From'] = sender
    msg['To'] = ', '.join(recipients)
    msg['Date'] = formatdate(localtime=True)
    msg['Subject'] = subject

    part = MIMEText(body, 'html')
    msg.attach(part)

    if object_type is not None and object_type.startswith('image'):
        with open('/tmp/thumbnail.jpg', 'rb') as f:
            img = MIMEImage(f.read())
            img.add_header('Content-ID', '<thumbnail>')
            msg.attach(img)

    response = ses.send_raw_email(
        Source=sender,
        Destinations=recipients,
        RawMessage={'Data': msg.as_string()}
    )
Set up S3 Event Trigger:
Go to your Lambda function.
Under "Designer", click on "Add trigger".
Choose S3 from the list of services.
Configure the trigger to monitor the desired S3 bucket and event type (e.g., "All object create events").
Save the trigger.
Set up SES:
Verify sender and recipient email addresses in the SES console.
Request production access for your AWS SES account.
Deploy and Test:
Save your Lambda function.
Upload an object to your S3 bucket and check if you receive the email with the object details.
This should get you started with the implementation. Let me know if you need further assistance!


#Lamda Role creation

here are the details for the IAM role you need to create:

Create a New Role:
Go to the IAM console: https://console.aws.amazon.com/iam/
Click on "Roles" in the left sidebar.
Click on "Create role".
Choose the Lambda Service Role:
Choose "AWS service" as the trusted entity.
Select "Lambda" as the service that will use this role.
Click on "Next: Permissions".
Attach Policies:
Search for and attach the following policies:
AmazonS3ReadOnlyAccess (for reading from S3)
AmazonSESFullAccess (for sending emails via SES)
Name and Review:
Name your role, e.g., "lambda-s3-ses-role".
Add a description if you want.
Click on "Create role".
That's it! Now you have a role with the necessary permissions for your Lambda function. You can select this role when creating your Lambda function.



#Details of Lambda function code line by line:

python
Copy code
import json
import boto3
import os
import mimetypes
from email.mime.multipart import MIMEMultipart
from email.mime.text import MIMEText
from email.mime.application import MIMEApplication
from email.mime.image import MIMEImage
from email.utils import formatdate
import json: Import the JSON module to handle JSON data.
import boto3: Import the Boto3 library, which is the AWS SDK for Python.
import os: Import the os module to work with files and directories.
import mimetypes: Import the mimetypes module to determine the MIME type of the uploaded object.
Import various modules from the email.mime package to help construct the email.
python
Copy code
s3 = boto3.client('s3')
ses = boto3.client('ses')
Create AWS service clients for S3 and SES using Boto3.
python
Copy code
def lambda_handler(event, context):
    bucket_name = event['Records'][0]['s3']['bucket']['name']
    object_key = event['Records'][0]['s3']['object']['key']
    object_size = event['Records'][0]['s3']['object']['size']
    object_type = mimetypes.guess_type(object_key)[0]
Define the Lambda handler function lambda_handler, which is the entry point for the Lambda function.
Extract information about the uploaded object from the S3 event. This includes the bucket name, object key, size, and type.
python
Copy code
    if object_type is not None and object_type.startswith('image'):
        create_thumbnail(bucket_name, object_key)
Check if the uploaded object is an image. If it is, call the create_thumbnail function to create a thumbnail.
python
Copy code
    send_email(bucket_name, object_key, object_size, object_type)
Call the send_email function to send an email with details about the uploaded object.
python
Copy code
    return {
        'statusCode': 200,
        'body': json.dumps('Email sent successfully!')
    }
Return a response indicating that the email was sent successfully.
python
Copy code
def create_thumbnail(bucket_name, object_key):
    s3.download_file(bucket_name, object_key, '/tmp/image.jpg') # Change file extension if needed
    # Code to create thumbnail using PIL or any other library
    # Save thumbnail to /tmp/thumbnail.jpg
    s3.upload_file('/tmp/thumbnail.jpg', bucket_name, 'thumbnails/' + os.path.basename(object_key))
Define the create_thumbnail function to download the image, create a thumbnail, and upload it to the S3 bucket.
python
Copy code
def send_email(bucket_name, object_key, object_size, object_type):
    recipients = ['user1@example.com', 'user2@example.com'] # Add email addresses of users to receive email
    sender = 'your-email@example.com' # Change this to your email address
    subject = 'S3 Upload Notification'

    body = f"""\
    <html>
      <head></head>
      <body>
        <p>Details of the uploaded object:</p>
        <ul>
          <li><strong>S3 Uri:</strong> s3://{bucket_name}</li>
          <li><strong>Object Name:</strong> {object_key}</li>
          <li><strong>Object Size:</strong> {object_size} bytes</li>
          <li><strong>Object Type:</strong> {object_type}</li>
        </ul>
        <p>Thumbnail:</p>
        <img src="cid:thumbnail">
      </body>
    </html>
    """
Define the email body with details about the uploaded object and include an image if the uploaded object is an image.
python
Copy code
    msg = MIMEMultipart()
    msg['From'] = sender
    msg['To'] = ', '.join(recipients)
    msg['Date'] = formatdate(localtime=True)
    msg['Subject'] = subject

    part = MIMEText(body, 'html')
    msg.attach(part)

    if object_type is not None and object_type.startswith('image'):
        with open('/tmp/thumbnail.jpg', 'rb') as f:
            img = MIMEImage(f.read())
            img.add_header('Content-ID', '<thumbnail>')
            msg.attach(img)

    response = ses.send_raw_email(
        Source=sender,
        Destinations=recipients,
        RawMessage={'Data': msg.as_string()}
    )
Construct the email message using the MIME multipart format.
Attach the HTML body to the email message.
If the uploaded object is an image, attach the thumbnail to the email.
Use the send_raw_email method of the SES client to send the email.
That's the explanation of the Lambda function code. Let me know if you have any further questions!

User
Is s3 and ses created as service or object 
ChatGPT
In the provided Lambda function code:

python
Copy code
s3 = boto3.client('s3')
ses = boto3.client('ses')
s3 and ses are created as service clients using the boto3.client() method.

s3 is an S3 service client, which allows you to interact with Amazon S3 service.
ses is an SES service client, which allows you to interact with Amazon SES (Simple Email Service) service.
These are created as Boto3 service clients, which are objects that represent connections to AWS services. These clients provide methods that you can use to interact with the corresponding AWS services.


#Event handler explanation in the Lambda function in simple terms:

In Python, a function is a block of code that performs a specific task. An event handler is a function that gets executed when a particular event occurs.

In the Lambda function:

python
Copy code
def lambda_handler(event, context):
    bucket_name = event['Records'][0]['s3']['bucket']['name']
    object_key = event['Records'][0]['s3']['object']['key']
    object_size = event['Records'][0]['s3']['object']['size']
    object_type = mimetypes.guess_type(object_key)[0]
lambda_handler is the name of our event handler function. In AWS Lambda, this function gets called automatically whenever the Lambda function is triggered.

event is a parameter of the lambda_handler function. When the Lambda function is triggered, AWS Lambda passes event data to the handler. In this case, the event data contains information about the S3 event that triggered the Lambda function.
context is another parameter of the lambda_handler function. It provides runtime information about the Lambda function.
Inside the lambda_handler function:

We extract information about the uploaded object from the event parameter. For example, event['Records'][0]['s3']['bucket']['name'] gives us the name of the S3 bucket where the object was uploaded, event['Records'][0]['s3']['object']['key'] gives us the key (i.e., the path) of the uploaded object, and event['Records'][0]['s3']['object']['size'] gives us the size of the uploaded object.
We use mimetypes.guess_type(object_key)[0] to determine the MIME type of the uploaded object.
So, in simple terms, the lambda_handler function is like a watchman waiting for something to happen. When an object is uploaded to the S3 bucket, AWS Lambda automatically calls the lambda_handler function and passes information about the uploaded object to it. Then, the function extracts this information and performs tasks like creating a thumbnail and sending an email.






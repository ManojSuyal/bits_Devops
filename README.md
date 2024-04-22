# Cloud

AWS Lambda Function for Monitoring S3 Uploads
This AWS Lambda function monitors an S3 bucket for new uploads. When a user uploads data into the S3 storage, the function captures the details of the uploaded object and sends an email to select users at the end of the day with the following information:

S3 Uri
Object Name
Object Size
Object Type
Additionally, if the uploaded object is an image (JPEG, JPG, or PNG), the function creates a thumbnail and stores it in the same S3 URI.

Implementation
Lambda Function Code
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
Setting up the Lambda Function
Go to the AWS Lambda console.
Click on "Create function" and choose "Author from scratch".
Give your function a name, choose the runtime as Python 3.8, and select an existing role or create a new one with necessary permissions to access S3 and SES.
Copy and paste the provided code into the code editor.
Save the function.
Setting up S3 Event Trigger
Go to your Lambda function.
Under "Designer", click on "Add trigger".
Choose S3 from the list of services.
Configure the trigger to monitor the desired S3 bucket and event type (e.g., "All object create events").
Save the trigger.
Setting up SES
Go to the AWS SES console.
Verify sender and recipient email addresses.
Request production access for your AWS SES account.

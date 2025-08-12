# Event Announcement Website — AWS S3, API Gateway, Lambda, SNS

## 📌 Project Overview
Develop an event announcement website that allows users to:
- Subscribe to event notifications via email.
- View a list of events.
- Create new events through a form.

**Frontend Hosting (S3)**  
Upload the HTML, CSS, and `events.json` files to S3 and enable static hosting to access the website URL.

**Backend API (API Gateway + Lambda + SNS)**  
Set up an API Gateway to handle backend processing for:
- Creating new events on `/create-event`
- Adding subscribers on `/subscribe`

**SNS + Lambda Integration**  
- Subscription Lambda adds new subscriber emails to the SNS topic.
- Event Registration Lambda updates `events.json` in S3 with new event details submitted from the website form and sends notifications via SNS.
  
<img width="764" height="490" alt="Screenshot 2025-08-12 at 1 53 00 AM" src="https://github.com/user-attachments/assets/b3c42f45-693b-4903-83fe-32c99a132bef" />

---

## 2.2 Set up Frontend Hosting with S3

### Steps to be Performed 👩‍💻
1. Create the website frontend files
2. Create an S3 bucket
3. Upload the Website files
4. Enable Static Hosting
5. Configure bucket policy for public access
6. Verify the website URL

### 1️⃣ Create the Website Frontend Files
Download pre-built files (`index.html`, `styles.css`, `events.json`) from:  
https://github.com/Eweka01/Cloud-Engineering-Project/tree/main/intermediate/project1

**If not using Git:**
- Download ZIP from GitHub.
- Extract it locally.

**Files required:**
- `index.html` — Website structure, event list display, subscription button
- `styles.css` — Visual appearance and layout
- `events.json` — Stores event data (title, date, description)

Example JSON:
```json
[
  {
    "title": "AWS Workshop",
    "date": "2024-11-20",
    "description": "Join us for a hands-on AWS workshop covering S3, CloudFront, and SNS!"
  },
  {
    "title": "Web Dev Bootcamp",
    "date": "2024-12-05",
    "description": "A beginner-friendly bootcamp to get you started with web development."
  }
]
```

---

### 2️⃣ Create an S3 Bucket
1. Open AWS S3 Console → Create Bucket.
2. **Bucket Name:** Unique name (e.g., `event-announcement-website`).
3. **Region:** Same as backend services.
4. **Block Public Access:** Disable "Block all public access".
5. Click **Create bucket**.

---

### 3️⃣ Upload Website Files to S3
- Upload `index.html`, `styles.css`, `events.json`
- Enable **public-read**.

---

### 4️⃣ Enable Static Website Hosting
- Go to **Properties** tab in S3 bucket.
- Scroll to **Static Website Hosting** → Enable.
- **Index Document:** `index.html`.

---

### 5️⃣ Set Permissions for Public Access
**Bucket Policy:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::<YOUR-BUCKET-NAME>/*"
    }
  ]
}
```
Replace `<YOUR-BUCKET-NAME>`.

---

### 6️⃣ Verify the Website
- Copy **Bucket Website Endpoint**.
- Open in browser.

---

## 2.3 Integrate SNS Notifications and Lambda Functions

### Steps to be Performed 👩‍💻
1. Create an SNS topic
2. Create & test Subscription Lambda
3. Create & test Event Creation Lambda

### 1️⃣ Create SNS Topic
- Go to **SNS Console** → Create Topic → Type: Standard.
- Name: `EventAnnouncements`.
- Copy **Topic ARN**.

---

### 2️⃣ Subscription Lambda
**IAM Role Permissions:**
- `AmazonSNSFullAccess`
- `AWSLambdaBasicExecutionRole`

**Lambda Function (Python 3.12):**
```python
import json
import boto3

def lambda_handler(event, context):
    print("Event received:", json.dumps(event))
    if 'body' in event:
        body = event['body'] if isinstance(event['body'], dict) else json.loads(event['body'])
        email = body.get('email', None)

        if email:
            sns_client = boto3.client('sns')
            try:
                sns_client.subscribe(
                    TopicArn='enter-sns-topic-ARN',
                    Protocol='email',
                    Endpoint=email
                )
                return {
                    'statusCode': 200,
                    'body': json.dumps({'message': 'Subscription successful! Please check your email to confirm.'})
                }
            except Exception as e:
                return {'statusCode': 500, 'body': json.dumps({'error': str(e)})}
        else:
            return {'statusCode': 400, 'body': json.dumps({'error': 'Email not provided.'})}
    return {'statusCode': 400, 'body': json.dumps({'error': 'Invalid request format.'})}
```

**Test Event:**
```json
{
  "body": {
    "email": "user@example.com"
  }
}
```

---

### 3️⃣ Event Creation Lambda
**IAM Role Permissions:**
- `AmazonS3FullAccess`
- `AmazonSNSFullAccess`

**Lambda Function (Python 3.12):**
```python
import json
import boto3
from botocore.exceptions import ClientError

s3 = boto3.client('s3')
sns = boto3.client('sns')

bucket_name = 'your-bucket-name'
events_file_key = 'events.json'
sns_topic_arn = 'your-sns-topic-arn'

def lambda_handler(event, context):
    try:
        new_event = json.loads(event['body'])
        response = s3.get_object(Bucket=bucket_name, Key=events_file_key)
        events_data = json.loads(response['Body'].read().decode('utf-8'))
        events_data.append(new_event)

        s3.put_object(
            Bucket=bucket_name,
            Key=events_file_key,
            Body=json.dumps(events_data, indent=2),
            ContentType='application/json'
        )

        message = f"New Event: {new_event['title']} on {new_event['date']}\n{new_event['description']}"
        sns.publish(TopicArn=sns_topic_arn, Message=message, Subject="New Event Announcement")

        return {'statusCode': 200, 'body': json.dumps({'message': 'Event created successfully!'})}
    except Exception as e:
        return {'statusCode': 500, 'body': json.dumps({'message': str(e)})}
```

**Test Event:**
```json
{
  "httpMethod": "POST",
  "body": "{"title":"Tech Meetup","date":"2024-12-01","description":"A gathering of tech enthusiasts!"}"
}
```

---

## 2.4 Setup, Test & Deploy API Gateway

1. Create REST API → Name: `EventManagementAPI` → Type: Regional.
2. Create `/subscribe` POST method → Integrate with `SubscribeToSNSFunction`.
3. Create `/create-event` POST method → Integrate with `CreateEventFunction`.
4. Enable CORS for both methods.
5. Add mapping templates for `/subscribe`:
```json
{
  "body": $input.json('$')
}
```
6. Deploy API to Stage: `dev`.

Example Endpoints:
```
/subscribe → https://<api-id>.execute-api.<region>.amazonaws.com/dev/subscribe
/create-event → https://<api-id>.execute-api.<region>.amazonaws.com/dev/create-event
```

---

## 2.5 Test and Finalize

### Update Frontend
In `index.html`, replace:
```javascript
fetch('<YOUR_SUBSCRIBE_API_ENDPOINT>', {...})
fetch('<YOUR_CREATE_EVENT_API_ENDPOINT>', {...})
```

### Upload Updated Files to S3
- Replace `index.html` in bucket.

### Test Website
- Subscribe → Confirm email.
- Create Event → Check S3 `events.json` updated and SNS email received.

---

## Cleanup 🗑️
1. Delete S3 bucket.
2. Delete Lambda functions.
3. Delete API Gateway API.
4. Delete SNS topic.
5. Remove IAM roles.

check out my medium for full detailed walk through 
https://medium.com/@oseweka1/cloud-powered-event-alerts-building-a-serverless-event-announcement-platform-with-aws-s3-lambda-3d23b230fc14

---

**⏱ Estimated Time:** 2-3 hours  
**💰 Cost:** Free Tier Eligible

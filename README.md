# DevOps Playground May 2021
In this session, you’ll learn how to write AWS Lambda functions in Python to interact with S3 and DynamoDB. We will be utilising the SDK for Python, known as Boto3. With Boto3, developers can create, configure, and manage AWS services through code. 

During this playground you will:
- Set up AWS Lambda to interact with other AWS services such as S3, DynamoDB, and CloudWatch
- Set up a trigger on S3 invoking the Lambda to take a CSV file and parse it to DynamoDB
- Configure permissions to allow the services to interact
- Utilise AWS CloudWatch
*************************
 1. **_Creating the S3 Bucket_** 

    S3 buckets are global and do not require a region. Each bucket name must be unique. 

    - Navigate to S3
    - Click **Create Bucket**
    - Name Bucket
        - S3 is global. They need to be unique
        - Naming convection: "playground-s3-*-panda"
        - Example: "playground-s3-silly-panda"
    - Block all public access
    - Create Bucket
    
    As you can see, the bucket is empty

2. **Create IAM role for Lambda
IAM role needs lambda, s3, dynamnodb and cloudwatch logs permissions**

    - Navigate to **IAM** (Global region)
    - Click on **Roles** in left hand side
    - Click **Create Role**
    - Click **Lambda** Under _Use Cases_
    - Next Permissions
    - In the search Bar type in _S3_
        - Select **playground-s3**
    - In the search bar type in _Dynamo_
        - Select **playground-dynamodb**
    - In the search bar type in _CloudWatch_
        - Select **playground-cloudwatch**
    - Next **Tags**. This is good practice! Add tags
        - Name: < your name >
    - Name Role - "playground-role-*-panda"
        - example "playground-role-silly-panda"
    
    Check to make sure all 3 policies are there
    
3. A **_Create Lambda Function_**
    - Navigate to Lambda
    - Make sure you are in the correct region < eu-west-2 >
    - Click **Create Function**
    - **Author From Scratch**
    - Function name- "playground-lambda-*-panda" (example "playground-lambda-silly-panda")
    - **Python 3.8**
    - Permissions
        - Use an existing role
        -  Find your role (playground-role-*-panda) 
    - **Create function**

    Check out that confirmation!

    
    _**3.B Adding a Trigger**_

    This step enables the Lambda function to be triggered by the event of uploading an item into our S3 bucket

    - Design- **Add Trigger**
    - Search- _S3_
    - Bucket- Search for the S3 bucket you created 
        - (playground-s3-*-panda)
    - Event Type- All 
    - Suffix- .csv
    - Prefix- (optional)
    - Click **Add**
    
Look for the confirmation. You can check the s3 details to make sure it is enabled.



4. _**Code to connect the Lambda function to the S3 bucket + testing our code**_

- Copy the code below into **lambda_function.py**:
```python
import json
import csv
import boto3

def lambda_handler(event, context):
    region = 'eu-west-2'
    record_list = []

    try:
        s3 = boto3.client('s3')  
        bucket = event['Records'][0]['s3']['bucket']['name']
        key = event['Records'][0]['s3']['object']['key']

        print('\nBucket: ', bucket, 'Key: ',key)

    except Exception as e:
        print(str(e))

    return {
        'statusCode':200,
        'body': json.dumps('CSV to DynamoDB Success!')
    }
```
- Click **Deploy**
- Click **Test**
- Navigate to the Test tab 
    - Event Template-Search 's3 put' (this shows the structure for the test)
    - Event name - "playgroundcsvtest"
    - Look for the S3 bucket
        - change bucket name to {name of bucket}
        - change object key to {name of file}
    - Create
    - Test again

5. Triggering the Lambda with a S3 upload.
- First let's add some code to read through the rows and print the items out so we can see it in the logs.
```python
import json
import csv
import boto3

def lambda_handler(event, context):
    region = 'eu-west-2'
    record_list = []

    try:
        s3 = boto3.client('s3') 
        bucket = event['Records'][0]['s3']['bucket']['name']
        key = event['Records'][0]['s3']['object']['key']

        print('\nBucket: ', bucket, '\nKey:',key)

        csv_file = s3.get_object(Bucket = bucket, Key= key)

        record_list = csv_file['Body'].read().decode('utf-8').split('\n')
        
        csv_reader = csv.reader(record_list, delimiter=',',quotechar='"')

        for row in csv_reader:
            movie_id = row[0]
            movie = row[1]
            title = row[2]
            year = row[3]

            print('\nMovie_ID: ', movie_id, '\nMovie: ', movie, '\nTitle: ', title, '\nYear: ', year)

    except Exception as e:
        print(str(e))

    return {
        'statusCode':200,
        'body': json.dumps('CSV to DynamoDB Success!')
    }
```

- Deploy
- Nagivate to S3 and upload your file to the S3 bucket.
- Test
- Check the logs to make sure it works! You should see the contents of the file in the output. 
To see logs navigate to **Cloudwatch** -> **Log groups** -> **/aws/lambda/playground-*-panda** / replace * with your adjective.

6. Create the DynamoDB Table
    - Navigate to DynamoDB in a new tab. Important! Make sure you are in the same region as your Lambda Function.
    - Click **Create table**.
    - Enter "playground-db-*-panda" for the **Table name**. (example: "playground-db-silly-panda")
    - Enter `movie_id` for the **Partition key** and select `Number` for the key type. 
    - Add tags because it's good practice! name: jillian, purpose: playground
    - Leave the **Use default settings** box checked and choose **Create**.
    - Click on the tabs on top just to check there are no items. After we upload, the items will appear here

7. Add code to connect the DynamoDB to the Lambda Function

```python
import json
import csv
import boto3

def lambda_handler(event, context):
    region = 'eu-west-2'
    record_list = []

    try:
        s3 = boto3.client('s3') 

        dynamodb = boto3.client('dynamodb', region_name = region)

        bucket = event['Records'][0]['s3']['bucket']['name']
        key = event['Records'][0]['s3']['object']['key']

        print('\nBucket: ', bucket, '\nKey:',key)

        csv_file = s3.get_object(Bucket = bucket, Key= key)

        record_list = csv_file['Body'].read().decode('utf-8').split('\n')

        
        csv_reader = csv.reader(record_list, delimiter=',',quotechar='"')

        for row in csv_reader:
            movie_id = row[0]
            movie = row[1]
            title = row[2]
            year = row[3]

            print('\nMovie_ID: ', movie_id, '\nMovie: ', movie, '\nTitle: ', title, '\nYear: ', year)

            add_to_db = dynamodb.put_item(
                TableName = 'playground-db-jillian',
                Item = {
                    'movie_id' : {'N':str(movie_id)},
                    'movie' : {'S':str(movie)},
                    'title' : {'S':str(title)},
                    'year' : {'N':str(year)},
                })
            print('\nSuccessfully added the records to the DynamoDB Table!\n')

    except Exception as e:
        print(str(e))

return {
    'statusCode':200,
    'body': json.dumps('CSV to DynamoDB Success!')
}
```


8. Check DynamoDB to make sure your items are there. You can also check out CLoudWatch and look through the logs. 



This concludes our lab today!
# DevOps Playground May 2021
In this session, you’ll learn how to write AWS Lambda functions in Python to interact with S3 and DynamoDB. We will be utilising the SDK for Python, known as Boto3. With Boto3, developers can create, configure, and manage AWS services through code. 

During this playground you will:
- Set up AWS Lambda to interact with other AWS services such as S3, DynamoDB, and CloudWatch
- Set up a trigger on S3 invoking the Lambda to take a CSV file and parse it to DynamoDB
- Configure permissions to allow the services to interact
- Utilise AWS CloudWatch
*************************
 1. **_Creating the S3 Bucket_** 

    S3 is global and does not require a region. Each bucket name must be unique. 

    - Navigate to S3
    - Click **Create Bucket**
    - Name Bucket
        - Naming convection: "playground-s3-*-panda"
        - Example: "playground-s3-silly-panda"
    - Block all public access
    - Add **Tags**
        - Name: Jillian (but add your name)
        - Purpose: Playground
    - Create Bucket
    
    As you can see, the bucket is empty

2. **_Create IAM role for Lambda. 
IAM role needs Lambda, S3, DynamnoDB and Cloudwatch logs permissions._**


    - Navigate to **IAM** (Global region) (Open in new tab)
    - Click to **Roles** on left hand side
    - Click **Create Role**
    - Click **Lambda** Under _Use Cases_
    - Next Permissions
    - In the search bar type in _playground_
        - Select **playground-s3**
        - Select **playground-dynamodb**
        - Select **playground-cloudwatch**
    - Next **Tags**. This is good practice! Add tags
        - Name: Beyonce (but add your name)
        - Purpose: Playground
    - **Next: Review**
    - Name Role - "playground-role-*-panda"
        - example "playground-role-silly-panda"
    - Create Role
    
    Check to make sure all 3 policies are there
    
3. A) **_Create Lambda Function_**
    - Navigate to Lambda (Open in new tab)
    - Make sure you are in the correct region < eu-west-2 >
    - Click **Create Function**
    - **Author From Scratch**
    - Function name- "playground-lambda-*-panda" 
        - (example "playground-lambda-silly-panda")
    - **Python 3.8**
    - Permissions
        - Use an existing role
        -  Find your role (playground-role-*-panda) 
    - **Create function**

    Check out that confirmation!

    
    _**3.B) Adding a Trigger**_

    This step enables the Lambda function to be triggered by the event of uploading an item into our S3 bucket

    - Click **Add Trigger**
    - Search- _S3_
    - Bucket- Search for the S3 bucket you created 
        - (playground-s3-*-panda)
    - Event Type- All 
    - Prefix- (optional)
    - Suffix- .csv
    - Click **Add**
    
Look for the confirmation. You can check the S3 details to make sure it is enabled.



4. _**Code to connect the Lambda function to the S3 bucket + Mocking an upload**_

- Copy the code below into **lambda_function.py**:

**Version 1**
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
            'statusCode': 500,
            'body': json.dumps('Oh no, something went wrong!')
        }
    else:
        return {
            'statusCode': 200,
            'body': json.dumps('Success!')
        }
```
- Click **Deploy** (this saves the code)
- **Test** > Configure Test Event
- Event template
    - Search _S3_ 
    - Select _Amazon S3 Put_
- Event name
    - playgroundcsvtest
- Create
- Click **Test** 
- Check the *Execution Results* to see the results

**Uploading **the test** CSV file to S3 and checking CloudWatch**
- Navigate back to S3 (open in new tab if not there already)
- Search for your bucket
- Upload the **playground-test.csv** file to your bucket 
    - Make sure you upload **playground-test.csv** and **not** playground-marvel.csv
    - Check to make sure the upload is successful

- Navigate back to *CloudWatch* (open in a new tab)
- Click on **Log groups** on the left hand side
- Click on your log group
- Select the latest entry under **LogStream**
- Check the logs for your **bucket name** and the **CSV file called playground-test.csv** 

5. _**Reading the CSV file with Python using boto3.**_
- Navigate back to your Lambda Function and copy this code into lambda_function.py

**Version 2**
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

        print('Bucket: ', bucket, '\nKey:',key)

        csv_file = s3.get_object(Bucket = bucket, Key = key)
        record_list = csv_file['Body'].read().decode('utf-8').split('\n')
        csv_reader  = csv.reader(record_list, delimiter = ',', quotechar = '"')

        for row in csv_reader:
            movie_id = row[0]
            movie    = row[1]
            title    = row[2]
            year     = row[3]

            print('\nMovie_ID: ', movie_id, '\nMovie: ', movie, '\nTitle: ', title, '\nYear: ', year)

    except Exception as e:
        print(str(e))
        
        return {
            'statusCode': 500,
            'body': json.dumps('Oh no, something went wrong!')
        }
    else:
        return {
            'statusCode': 200,
            'body': json.dumps('Success!')
        }
```
- Click **Deploy** (this saves the code)

**To mock test an upload**
- Click on **Configure Test Events** in the **Test** dropdown 
- Make sure you are in your previously saved test 
- Under "S3" replace "example-bucket" with the name of your bucket
    - "playground-s3-silly-panda"
- Under "object" replace "test/key" with the name of the  **test** csv file 
    - playground-test.csv
- Save
- Click on **Test** to simulate a CSV upload ( trigger )
- Check the _Execution Results_ to see your bucket name and the name of the test CSV file
 

6. _**Create the DynamoDB Table**_
    - Navigate to DynamoDB in a new tab. **Important! Make sure you are in the same region as your Lambda Function.**
    - Click **Create table**.
    - Enter "playground-db-*-panda" for the **Table name**. (example: "playground-db-silly-panda")
    - Enter < movie_id > for the **Partition key** and select < Number > for the key type. 
    - Add tags because it's good practice! 
        - Name: Elon (but put your name)
        - Purpose: Playground
    - Leave the **Use default settings** box checked and choose **Create**.
    - Click on the **Items** tab on top just to check there are no items. After we upload, the items will appear here

7. _**Add code to connect the DynamoDB to the Lambda Function**_

- Navigate back to Lambda
- Click on lambda_function.py and paste the code below in this file. 

**Version 3**
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
        
        print('Bucket: ', bucket, '\nKey:',key)
        
        csv_file = s3.get_object(Bucket = bucket, Key= key)
        record_list = csv_file['Body'].read().decode('utf-8').split('\n')
        csv_reader = csv.reader(record_list, delimiter = ',', quotechar = '"')
        
        for row in csv_reader:
            movie_id = row[0]
            movie    = row[1]
            title    = row[2]
            year     = row[3]
            
            print('\nMovie_ID: ', movie_id, '\nMovie: ', movie, '\nTitle: ', title, '\nYear: ', year)
            
            add_db = dynamodb.put_item(
                TableName = 'playground-db-silly-panda',
                Item = {
                    'movie_id' : {'N' : str(movie_id)},
                    'movie'    : {'S' : str(movie)},
                    'title'    : {'S' : str(title)},
                    'year'     : {'N' : str(year)}
                }
            )
            
            print('\nSuccessfully added the records to the DynamoDB Table!')
    
    except Exception as e:
        print(str(e))
        
        return {
            'statusCode': 500,
            'body': json.dumps('Oh no, something went wrong!')
        }
    else:
        return {
            'statusCode': 200,
            'body': json.dumps('Success!')
        }
```
- Change the TableName on Line 30 to the name of your table
- Click **Deploy** 
- Navigate back to S3
- Upload **playground-marvel.csv** 

8. Navigate back to DynamoDB and check the items. The items in the CSV file should be there!

Hooray! This concludes our lab today! 

Thanks for joining the DevOps Playground!

# Serverless-project

Overview And High Level Design
Here I wanted to test a serverless API using API Gateway, Lambda, and DynamoDB.
First, I created one resource (DynamoDBManager) and defined one method (POST) on it. The method is backed by a Lambda function (LambdaFunctionOverHttps). I did so for calling the API through an HTTPS endpoint; then Amazon API Gateway will invoke the Lambda function.

The POST method on the DynamoDBManager resource supports the following DynamoDB operations:

Create, update, and delete an item.
Read an item.
Scan an item.
Other operations (echo, ping), not related to DynamoDB, that can be used for testing.
The request payload sent in the POST request identifies the DynamoDB operation and provides necessary data. For example:

The following is a sample request payload for a DynamoDB create item operation:

{
    "operation": "create",
    "tableName": "lambda-apigateway",
    "payload": {
        "Item": {
            "id": "1",
            "name": "Bob"
        }
    }
}
The following is a sample request payload for a DynamoDB read item operation:

{
    "operation": "read",
    "tableName": "lambda-apigateway",
    "payload": {
        "Key": {
            "id": "1"
        }
    }
}
Setup
I created a Lambda IAM Role. Created the execution role that gives the function permission to access AWS resources.

I created the execution role with the following properties.
Trusted entity – Lambda.
Role name – lambda-apigateway-role.
Permissions – Custom policy with permission to DynamoDB and CloudWatch Logs. This custom policy has the permissions that the function needs to write data to DynamoDB and upload logs.
{
"Version": "2012-10-17",
"Statement": [
{
  "Sid": "Stmt1428341300017",
  "Action": [
    "dynamodb:DeleteItem",
    "dynamodb:GetItem",
    "dynamodb:PutItem",
    "dynamodb:Query",
    "dynamodb:Scan",
    "dynamodb:UpdateItem"
  ],
  "Effect": "Allow",
  "Resource": "*"
},
{
  "Sid": "",
  "Resource": "*",
  "Action": [
    "logs:CreateLogGroup",
    "logs:CreateLogStream",
    "logs:PutLogEvents"
  ],
  "Effect": "Allow"
}
]
}
Then, I created the Lambda Function
I selected "Author from scratch". Used name LambdaFunctionOverHttps , selected Python 3.7 as Runtime. Under Permissions, selected "Use an existing role", and selected lambda-apigateway-role that I created, from the drop down


I replaced the boilerplate coding with the following code snippet.
Python Code

from __future__ import print_function

import boto3
import json

print('Loading function')


def lambda_handler(event, context):
    '''Provide an event that contains the following keys:

      - operation: one of the operations in the operations dict below
      - tableName: required for operations that interact with DynamoDB
      - payload: a parameter to pass to the operation being performed
    '''
    #print("Received event: " + json.dumps(event, indent=2))

    operation = event['operation']

    if 'tableName' in event:
        dynamo = boto3.resource('dynamodb').Table(event['tableName'])

    operations = {
        'create': lambda x: dynamo.put_item(**x),
        'read': lambda x: dynamo.get_item(**x),
        'update': lambda x: dynamo.update_item(**x),
        'delete': lambda x: dynamo.delete_item(**x),
        'list': lambda x: dynamo.scan(**x),
        'echo': lambda x: x,
        'ping': lambda x: 'pong'
    }

    if operation in operations:
        return operations[operation](event.get('payload'))
    else:
        raise ValueError('Unrecognized operation "{}"'.format(operation))
Lambda Code

Then, I tested the Lambda Function
As I haden't created DynamoDB and the API yet, I used a sample echo operation. The function should output whatever input I pass.

The field "operation" dictates what the lambda function will perform. In this case, it'd simply return the payload from input event as output.
{
    "operation": "echo",
    "payload": {
        "somekey1": "somevalue1",
        "somekey2": "somevalue2"
    }
}

At this point I created DynamoDB table and an API using the lambda as backend.

DynamoDB Table
I created the DynamoDB table that the Lambda function uses with the following settings.
Table name – lambda-apigateway
Primary key – id (string)

Then, I create API selecting "Build" for REST API as "DynamoDBOperations".

Each API is collection of resources and methods that are integrated with backend HTTP endpoints, Lambda functions, or other AWS services. Typically, API resources are organized in a resource tree according to the application logic.
I created a resource ad "DynamoDBManager" in the Resource Name and a POST Method for the API. I selected "LambdaFunctionOverHttps" function that I created earlier. 

Now the API-Lambda integration is done!

Lastly, the API deplying is need.

Finally everything is set to run the solution! To invoke the API endpoint,the endpoint url is needed. 

Running the solution
The Lambda function supports using the create operation to create an item in the DynamoDB table. To request this operation, I used the following JSON:
{
    "operation": "create",
    "tableName": "lambda-apigateway",
    "payload": {
        "Item": {
            "id": "1234ABCD",
            "number": 5
        }
    }
}
To execute the API from local machine, I used Postman and Curl command.

To run this from Postman, I selected "POST" , pasted the API invoke url. Then under "Body" selected "raw" and paste the above JSON. Click "Send". API executed and returned "HTTPStatusCode" 200.
Execute from Postman

To run this from terminal using Curl:
$ curl -X POST -d "{\"operation\":\"create\",\"tableName\":\"lambda-apigateway\",\"payload\":{\"Item\":{\"id\":\"1\",\"name\":\"Bob\"}}}" https://$API.execute-api.$REGION.amazonaws.com/prod/DynamoDBManager
To validate that the item is indeed inserted into DynamoDB table, it can be seen in the Dynamo console.

Dynamo Item

To get all the inserted items from the table, I used the "list" operation of Lambda using the same API.
{
    "operation": "list",
    "tableName": "lambda-apigateway",
    "payload": {
    }
}

Now, I have successfully created a serverless API using API Gateway, Lambda, and DynamoDB!

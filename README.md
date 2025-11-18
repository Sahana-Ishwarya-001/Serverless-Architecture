# Serverless-Architecture

Most serverless projects teach you *what* to build.

In my projects, I cared more about *why* I built things a certain way and what patterns actually scale in production.

So instead of a generic tutorial, this repository explains how I’ve used this exact workflow (API Gateway → Lambda → DynamoDB) in real systems, and why this project mirrors a practical, production-aligned serverless architecture.

<img width="864" height="193" alt="image" src="https://github.com/user-attachments/assets/b88c1f3b-0f11-4503-862b-c0d143f4ef45" />

---

## **High-Level Design**

A lot of serverless APIs build on following the familiar pattern:

* **Amazon API Gateway** exposes an HTTPS endpoint.
* A **POST** method receives a JSON payload describing the action.
* API Gateway triggers a **Lambda function**.
* Lambda performs one of several DynamoDB operations depending on the request.

This project uses exactly the same model.

### **Resource Design**

I created a single resource:

```
/DynamoDBManager
```

with a single method:

```
POST
```

Why POST method?

When you’re building internal utilities or microservices that interact with DynamoDB, POST gives you flexibility to send structured payloads for:

* create
* read
* update
* delete
* scan
* and testing operations (echo, ping)

This “operation router” approach is extremely common in lightweight serverless backends.

### **Sample Requests**

A create request looks like this:

```json
{
  "operation": "create",
  "tableName": "lambda-apigateway",
  "payload": {
    "Item": {
      "id": "1",
      "name": "Samyu"
    }
  }
}
```

And a read request:

```json
{
  "operation": "read",
  "tableName": "lambda-apigateway",
  "payload": {
    "Key": {
      "id": "1"
    }
  }
}
```

This pattern has worked extremely well in my production systems — no multiple API routes, no extra Lambdas, easy to extend, and predictable payloads.

---

# **Project Setup**

When I build serverless APIs, I always start with security and least privilege, not code.
This project follows that same principle.

---

## **1. Create a Custom IAM Policy (Least Privilege)**

In real-world workloads, giving Lambda full DynamoDB access is a no-go.
So I created a minimal policy:

* Only the CRUD and read operations needed.
* Only the ability to write logs (CloudWatch).
* Nothing more

Policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
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
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Effect": "Allow",
      "Resource": "*"
    }
  ]
}
```

I named it: **lambda-custom-policy**.

---

## **2. Create the Lambda Execution Role**

This is the execution role the function will assume.

I attached the above custom policy, named the role:

```
lambda-apigateway-role
```

This mirrors how I structure production: every Lambda has a dedicated role with only the permissions it needs.

---

## **3. Create the Lambda Function**

The function name:

```
LambdaFunctionOverHttps
```

Runtime:

```
Python 3.13
```

I attached the IAM role we created earlier.

Then I replaced the default code with:

```python
from __future__ import print_function
import boto3
import json

def lambda_handler(event, context):
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
        raise ValueError(f'Unrecognized operation "{operation}"')
```

### Why this "router Lambda" design works:

* Adding new operations takes seconds.
* Payload structure stays predictable.
* API Gateway doesn’t need multiple routes.
* Low latency, low cost.
* Extremely flexible for internal tools

---

## **4. Test the Lambda First**

Before wiring in DynamoDB or API Gateway, I always verify the Lambda locally in the console.
The “echo” operation is perfect for this.

Test event:

```json
{
  "operation": "echo",
  "payload": {
    "somekey1": "somevalue1",
    "somekey2": "somevalue2"
  }
}
```

If the output matches the input —> you know your Lambda wiring is good.

---

# **Create DynamoDB Table**

Table name:

```
lambda-apigateway
```

Primary key:

```
id (String)
```

Later, I confirmed items inserted via API show up here.

---

# **Create API Gateway**

This is where the architecture starts to feel like production.

API name:

```
DynamoDBOperations
```

I added the resource:

```
/DynamoDBManager
```

Then created the **POST** method and connected it to the Lambda.

This is exactly how most micro-APIs start:

* One resource
* One method
* Lambda handles the backend logic

---

# **Deploy the API**

I deployed to Prod stage.

```
Stage: Prod
```

After deployment, API Gateway provides an Invoke URL, which becomes the main endpoint for all operations.

---

# **Running the Solution**

### Create item:

```json
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
```

### Test it using Postman or cURL:

```bash
curl -X POST -d "{\"operation\":\"create\",\"tableName\":\"lambda-apigateway\",\"payload\":{\"Item\":{\"id\":\"1\",\"name\":\"Samyu\"}}}" https://$API.execute-api.$REGION.amazonaws.com/prod/DynamoDBManager
```

Then I validated the insert directly from the DynamoDB console.

To list all items:

```json
{
  "operation": "list",
  "tableName": "lambda-apigateway",
  "payload": {}
}
```

This endpoint became a mini DynamoDB console for me — incredibly handy for internal tooling and admin utilities.

---

# **Cleanup**

Since this is a github project, I removed:

* DynamoDB table
* Lambda function
* API Gateway API

In real deployments, I tag everything and clean up automatically with IaC (Terraform, CDK, CloudFormation).
But for console-based labs, manual deletion works fine.

---

# **Final Thoughts**

This project isn’t just a “how-to”.
It mirrors real internal APIs used across enterprise serverless architecture.

You’re learning patterns that matter:

* Single-endpoint operation routers
* Stateless Lambda design
* Least-privilege IAM
* API Gateway + Lambda integration
* DynamoDB CRUD operations via JSON payloads

If you’re leveling up your serverless architecture skills, this is one of the best hands-on exercises to anchor the concepts.

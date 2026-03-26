# Day 36 — AWS Lambda, API Gateway & CloudFormation Language Extensions

Two tasks covering serverless compute with Lambda and API Gateway, and advanced CloudFormation templating using the AWS Language Extensions transform with `Fn::ForEach` and dynamic resource creation.

---

## Table of Contents

- [Task 1 — Lambda Function + API Gateway](#task-1--lambda-function--api-gateway)
- [Task 2 — CloudFormation Language Extensions (Fn::ForEach)](#task-2--cloudformation-language-extensions-fnforeach)
- [CloudFormation Language Extensions — Concepts](#cloudformation-language-extensions--concepts)
- [Fn::ForEach — Deep Dive](#fnforeach--deep-dive)
- [Fn::Length — Deep Dive](#fnlength--deep-dive)
- [Bugs and Gotchas](#bugs-and-gotchas)
- [Key Concepts Learned](#key-concepts-learned)

---

## Task 1 — Lambda Function + API Gateway

### What this task does

Deploys a Python Lambda function behind an API Gateway REST API. The API Gateway exposes a single resource `/myresource` that accepts any HTTP method (`GET`, `POST`, `PUT`, `DELETE`) and routes it to the Lambda function. The Lambda function inspects the HTTP method and returns a different response for each one. Every request and response is logged to CloudWatch.

After deployment the endpoint is tested using Postman.

### Architecture

```
Postman / Browser
       │
       ▼
API Gateway (REST API)        — regional endpoint
       │  resource: /myresource
       │  method: ANY
       │  integration: AWS_PROXY
       ▼
Lambda Function               — Python 3.12
       │  reads httpMethod from event
       │  returns JSON response
       ▼
CloudWatch Logs               — full request + response logged
```

### Resources Created

| Resource | Type | Details |
|---|---|---|
| LambdaExecutionRole | IAM Role | Allows Lambda to write CloudWatch logs |
| LambdaFunction | Lambda Function | Python, handles GET/POST/PUT/DELETE |
| ApiGateway | REST API | Regional endpoint |
| MyApiResource | API Resource | Path: `/myresource` |
| MyApiMethod | API Method | ANY method, AWS_PROXY integration |
| LambdaPermissionForApiGateway | Lambda Permission | Allows API GW to invoke Lambda |
| Deployment | API Deployment | Stage: `prod` |

---

### How API Gateway Integrates with Lambda — AWS_PROXY

The integration type `AWS_PROXY` means API Gateway passes the entire HTTP request as-is to Lambda inside an `event` object. Lambda is fully responsible for building the response — API Gateway does no mapping or transformation.

```
HTTP Request arrives at API GW
       │
       │  AWS_PROXY wraps it into:
       ▼
{
  "httpMethod": "GET",
  "path": "/myresource",
  "queryStringParameters": { "name": "alice" },
  "headers": { "Content-Type": "application/json" },
  "body": null
}
       │
       ▼
Lambda receives the event dict
       │
       ▼
Lambda returns:
{
  "statusCode": 200,
  "headers": { "Content-Type": "application/json" },
  "body": "{\"message\": \"GET request successful!\"}"
}
       │
       ▼
API GW forwards that response directly to the client
```

Without `AWS_PROXY` you would need to manually configure request/response mapping templates in API Gateway. `AWS_PROXY` skips all of that and hands control entirely to the Lambda function.

---

### Lambda Function — What it Does

```
1. Log the full incoming request to CloudWatch
      - timestamp, httpMethod, path, query params, headers, body

2. Parse the request body safely
      - json.loads() wrapped in try/except to handle malformed JSON

3. Route on httpMethod:
      GET     → return message + query params + timestamp
      POST    → return message + parsed body + timestamp
      PUT     → return message + parsed body + timestamp
      DELETE  → return message + timestamp
      other   → return 405 Method Not Allowed

4. Log the outgoing response to CloudWatch
      - status code + response body

5. Return the Lambda proxy response format:
      {
        statusCode: int,
        headers: { Content-Type, Access-Control-Allow-Origin },
        body: JSON string
      }
```

### The Lambda Proxy Response Format

Lambda must return a specific structure for API Gateway to forward it correctly. If this structure is wrong, API Gateway returns a 502 Bad Gateway.

```python
return {
    "statusCode": 200,           # int — HTTP status code
    "headers": {                 # dict — response headers
        "Content-Type": "application/json",
        "Access-Control-Allow-Origin": "*"   # CORS header
    },
    "body": json.dumps(response_body)   # string — must be a string, not a dict
}
```

> **Important:** `body` must be a **string**, not a dict. Returning a dict directly causes a 502. Always call `json.dumps()` on the response body before returning.

---

### IAM Role — Why These Permissions

```yaml
Action:
  - logs:CreateLogGroup
  - logs:CreateLogStream
  - logs:PutLogEvents
Resource: "arn:aws:logs:*:*:*"
```

Every Lambda function needs permission to write to CloudWatch Logs. Without these three actions the function runs but produces no logs — debugging becomes impossible. `logs:CreateLogGroup` creates the `/aws/lambda/FunctionName` log group on first invocation. `logs:CreateLogStream` creates the log stream. `logs:PutLogEvents` writes the actual log lines.

---

### Lambda Permission Resource

```yaml
LambdaPermissionForApiGateway:
  Type: AWS::Lambda::Permission
  Properties:
    FunctionName: !Ref LambdaFunction
    Action: lambda:InvokeFunction
    Principal: apigateway.amazonaws.com
    SourceArn: !Sub 'arn:aws:execute-api:${AWS::Region}:${AWS::AccountId}:${ApiGateway}/*/*/myresource'
```

This resource is easy to overlook but critical. Without it, API Gateway cannot invoke the Lambda function and returns a 403. The `SourceArn` uses wildcards for stage (`*`) and method (`*`) so any stage and any HTTP method on `/myresource` can invoke the function.

---

### Testing with Postman

After stack deployment, get the endpoint from the stack Outputs:

```
https://<api-id>.execute-api.us-east-1.amazonaws.com/prod/myresource
```

**GET request**
```
Method : GET
URL    : https://.../prod/myresource?name=alice
```
Expected response:
```json
{
  "message": "GET request successful!",
  "query": { "name": "alice" },
  "timestamp": "2024-01-15T10:30:00.123456"
}
```

**POST request**
```
Method  : POST
URL     : https://.../prod/myresource
Headers : Content-Type: application/json
Body    : { "name": "alice", "age": 25 }
```
Expected response:
```json
{
  "message": "POST request successful!",
  "receivedData": { "name": "alice", "age": 25 },
  "timestamp": "2024-01-15T10:30:00.123456"
}
```

**DELETE request**
```
Method : DELETE
URL    : https://.../prod/myresource
```
Expected response:
```json
{
  "message": "DELETE request successful!",
  "timestamp": "2024-01-15T10:30:00.123456"
}
```

---

### Task 1 — Full Template

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: 'CloudFormation template that consist of the Lambda function and the API Gateway to trigger the Lambda function'

Resources:
  LambdaExecutionRole:
    Type: AWS::IAM::Role
    Properties:
      RoleName: 'MyLambdaExecutionRole'
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service: lambda.amazonaws.com
            Action: sts:AssumeRole
      Policies:
        - PolicyName: 'LambdaBasicExecutionPolicy'
          PolicyDocument:
            Version: '2012-10-17'
            Statement:
              - Effect: Allow
                Action:
                  - logs:CreateLogGroup
                  - logs:CreateLogStream
                  - logs:PutLogEvents
                Resource: "arn:aws:logs:*:*:*"

  LambdaFunction:
    Type: AWS::Lambda::Function
    Properties:
      FunctionName: 'MyLambdaFunction'
      Handler: index.lambda_handler
      Runtime: 'python3.12'
      Role: !GetAtt LambdaExecutionRole.Arn
      Code:
        ZipFile: |
          import json
          from datetime import datetime

          def lambda_handler(event, context):

              print("===== INCOMING REQUEST =====")
              print("Timestamp     :", datetime.utcnow().isoformat())
              print("HTTP Method   :", event.get("httpMethod"))
              print("Path          :", event.get("path"))
              print("Query Params  :", json.dumps(event.get("queryStringParameters")))
              print("Headers       :", json.dumps(event.get("headers")))
              print("Body          :", event.get("body"))
              print("============================")

              request_body = {}
              if event.get("body"):
                  try:
                      request_body = json.loads(event["body"])
                  except Exception as e:
                      print("Body parse error:", str(e))

              http_method = event.get("httpMethod", "")
              status_code = 200

              if http_method == "GET":
                  response_body = {
                      "message"   : "GET request successful!",
                      "query"     : event.get("queryStringParameters") or {},
                      "timestamp" : datetime.utcnow().isoformat()
                  }
              elif http_method == "POST":
                  response_body = {
                      "message"      : "POST request successful!",
                      "receivedData" : request_body,
                      "timestamp"    : datetime.utcnow().isoformat()
                  }
              elif http_method == "PUT":
                  response_body = {
                      "message"     : "PUT request successful!",
                      "updatedData" : request_body,
                      "timestamp"   : datetime.utcnow().isoformat()
                  }
              elif http_method == "DELETE":
                  response_body = {
                      "message"   : "DELETE request successful!",
                      "timestamp" : datetime.utcnow().isoformat()
                  }
              else:
                  status_code   = 405
                  response_body = { "error": "Method Not Allowed" }

              print("===== OUTGOING RESPONSE =====")
              print("Status Code:", status_code)
              print("Response   :", json.dumps(response_body))
              print("=============================")

              return {
                  "statusCode" : status_code,
                  "headers"    : {
                      "Content-Type"                : "application/json",
                      "Access-Control-Allow-Origin" : "*"
                  },
                  "body" : json.dumps(response_body)
              }
      Tags:
        - Key: 'Project'
          Value: 'DemoFunction'

  ApiGateway:
    Type: AWS::ApiGateway::RestApi
    Properties:
      Name: 'MyApiGateway'
      Description: 'API Gateway to trigger the Lambda function'
      EndpointConfiguration:
        Types:
          - REGIONAL

  MyApiResource:
    Type: AWS::ApiGateway::Resource
    Properties:
      RestApiId: !Ref ApiGateway
      ParentId: !GetAtt ApiGateway.RootResourceId
      PathPart: 'myresource'

  MyApiMethod:
    Type: AWS::ApiGateway::Method
    Properties:
      RestApiId: !Ref ApiGateway
      ResourceId: !Ref MyApiResource
      HttpMethod: 'ANY'
      AuthorizationType: 'NONE'
      Integration:
        Type: AWS_PROXY
        IntegrationHttpMethod: 'POST'
        Uri: !Sub 'arn:aws:apigateway:${AWS::Region}:lambda:path/2015-03-31/functions/${LambdaFunction.Arn}/invocations'
      MethodResponses:
        - StatusCode: '200'
          ResponseModels:
            application/json: 'Empty'
        - StatusCode: '400'
          ResponseModels:
            application/json: 'Empty'
        - StatusCode: '500'
          ResponseModels:
            application/json: 'Empty'

  LambdaPermissionForApiGateway:
    Type: AWS::Lambda::Permission
    Properties:
      FunctionName: !Ref LambdaFunction
      Action: lambda:InvokeFunction
      Principal: apigateway.amazonaws.com
      SourceArn: !Sub 'arn:aws:execute-api:${AWS::Region}:${AWS::AccountId}:${ApiGateway}/*/*/myresource'

  Deployment:
    Type: AWS::ApiGateway::Deployment
    DependsOn: MyApiMethod
    Properties:
      RestApiId: !Ref ApiGateway
      StageName: 'prod'

Outputs:
  ApiEndpoint:
    Description: "API Gateway endpoint URL"
    Value: !Sub 'https://${ApiGateway}.execute-api.${AWS::Region}.amazonaws.com/prod/myresource'
```

---

## Task 2 — CloudFormation Language Extensions (Fn::ForEach)

### What this task does

Uses the `AWS::LanguageExtensions` transform to dynamically create EC2 instances based on the environment type. In `dev` one instance is created. In `prod` two instances are created. The instance type and count both come from a `Mappings` block — no hardcoded resource blocks needed.

### What makes this different from a regular template

In a normal CloudFormation template every resource must be declared explicitly. If you want two EC2 instances you write two `AWS::EC2::Instance` blocks. If the count changes between environments you either duplicate code or use separate templates.

With `Fn::ForEach` and the Language Extensions transform, you write one resource block and CloudFormation expands it into N resources at deploy time based on a list you provide.

```
Without ForEach (hardcoded):          With ForEach (dynamic):
  EC2Instance1: ...                     Fn::ForEach::CreateInstances:
  EC2Instance2: ...                       - Instance
  EC2Instance3: ...  (prod only)          - ["1", "2"]   ← from Mappings
  EC2Instance4: ...  (prod only)          - EC2Instance&{Instance}: ...
```

### How the template works step by step

**Step 1 — Transform declaration**

```yaml
Transform: AWS::LanguageExtensions
```

This line tells CloudFormation to pre-process the template through the Language Extensions transform before deploying. Without it `Fn::ForEach` is not recognised and deployment fails.

**Step 2 — Parameters**

```yaml
Parameters:
  EnvType:
    Type: String
    AllowedValues: [dev, prod]

  AMIId:
    Type: AWS::SSM::Parameter::Value<AWS::EC2::Image::Id>
    Default: /aws/service/ami-amazon-linux-latest/amzn2-ami-hvm-x86_64-gp2
```

`EnvType` is the environment selector. `AMIId` automatically fetches the latest Amazon Linux 2 AMI from SSM Parameter Store — no hardcoded AMI IDs needed.

**Step 3 — Mappings**

```yaml
Mappings:
  InstanceTypeMap:
    dev:
      InstanceType: t3.micro
      InstanceCount: ["1"]
    prod:
      InstanceType: t3.small
      InstanceCount: ["1", "2"]
```

The `InstanceCount` list drives how many instances are created. `["1"]` = 1 instance. `["1", "2"]` = 2 instances. The values in the list (`"1"`, `"2"`) are used as suffixes in the resource name and instance tag.

**Step 4 — ForEach loop**

```yaml
'Fn::ForEach::CreateInstances':
  - Instance                     # loop variable name
  - !FindInMap [InstanceTypeMap, !Ref EnvType, InstanceCount]   # the list to iterate
  - 'EC2Instance&{Instance}':    # resource name template — &{} substitutes the loop variable
      Type: AWS::EC2::Instance
      Properties:
        ImageId: !Ref AMIId
        InstanceType: !FindInMap [InstanceTypeMap, !Ref EnvType, InstanceType]
        Tags:
          - Key: Name
            Value: !Sub "${EnvType}-instance-&{Instance}"
```

When `EnvType=dev` this expands to one resource: `EC2Instance1`
When `EnvType=prod` this expands to two resources: `EC2Instance1` and `EC2Instance2`

---

### Task 2 — Full Template

```yaml
AWSTemplateFormatVersion: "2010-09-09"
Transform: AWS::LanguageExtensions

Parameters:
  EnvType:
    Type: String
    AllowedValues:
      - dev
      - prod

  AMIId:
    Type: AWS::SSM::Parameter::Value<AWS::EC2::Image::Id>
    Default: /aws/service/ami-amazon-linux-latest/amzn2-ami-hvm-x86_64-gp2

Mappings:
  InstanceTypeMap:
    dev:
      InstanceType: t3.micro
      InstanceCount: ["1"]
    prod:
      InstanceType: t3.small
      InstanceCount: ["1", "2"]

Conditions:
  IsProd: !Equals [!Ref EnvType, "prod"]

Resources:
  'Fn::ForEach::CreateInstances':
    - Instance
    - !FindInMap
        - InstanceTypeMap
        - !Ref EnvType
        - InstanceCount
    - 'EC2Instance&{Instance}':
        Type: AWS::EC2::Instance
        Properties:
          ImageId: !Ref AMIId
          InstanceType: !FindInMap
            - InstanceTypeMap
            - !Ref EnvType
            - InstanceType
          Tags:
            - Key: Name
              Value: !Sub "${EnvType}-instance-&{Instance}"
            - Key: Environment
              Value: !Ref EnvType

Outputs:
  EnvironmentType:
    Description: "Deployed environment"
    Value: !Ref EnvType

  TotalInstances:
    Description: "Total instances deployed"
    Value: !If [IsProd, "2", "1"]
```

---

## CloudFormation Language Extensions — Concepts

### What is the AWS::LanguageExtensions Transform?

`AWS::LanguageExtensions` is a CloudFormation transform that adds new intrinsic functions and capabilities not available in standard CloudFormation. It is declared at the top of the template and pre-processes the template before CloudFormation evaluates it.

```yaml
AWSTemplateFormatVersion: "2010-09-09"
Transform: AWS::LanguageExtensions   # must be declared at template level
```

Without this declaration any Language Extensions function (`Fn::ForEach`, `Fn::Length`, `Fn::ToJsonString`) causes a deployment error.

### Functions added by AWS::LanguageExtensions

| Function | What it does |
|---|---|
| `Fn::ForEach` | Iterates over a list and replicates a template snippet for each item |
| `Fn::Length` | Returns the number of items in a list |
| `Fn::ToJsonString` | Converts an object or array to a JSON string |
| `Fn::FindInMap` (extended) | Adds default value support to the standard FindInMap |

### CAPABILITY_AUTO_EXPAND

Templates that use transforms require an additional capability flag:

```bash
aws cloudformation deploy \
  --template-file template.yaml \
  --stack-name my-stack \
  --capabilities CAPABILITY_AUTO_EXPAND CAPABILITY_IAM
```

`CAPABILITY_AUTO_EXPAND` explicitly acknowledges that the template uses a transform (macro) that expands the template before deployment. Without it the deployment fails with a `InsufficientCapabilities` error.

---

## Fn::ForEach — Deep Dive

### Syntax

`Fn::ForEach` has a specific three-element structure. The key name must start with `Fn::ForEach::` followed by a unique identifier (the loop name).

```yaml
'Fn::ForEach::UniqueLoopName':
  - VariableName          # element 1: name of the loop variable
  - [list, to, iterate]   # element 2: the list — can be hardcoded or from FindInMap/parameter
  - ResourceTemplate      # element 3: the template to replicate for each item
```

### Variable Substitution — &{} vs ${}

Inside a `Fn::ForEach` block there are **two different substitution syntaxes** and mixing them up is the most common mistake:

| Syntax | Used for | Works in |
|---|---|---|
| `&{VariableName}` | ForEach loop variable | Resource logical IDs, property values inside ForEach |
| `${VariableName}` | CloudFormation parameters/refs | Inside `!Sub` strings |

```yaml
# Correct usage
'EC2Instance&{Instance}':      # &{} in resource logical ID
  Type: AWS::EC2::Instance
  Properties:
    Tags:
      - Key: Name
        Value: !Sub "instance-&{Instance}"   # &{} also works inside !Sub
```

> **Critical rule:** Resource logical IDs (the key names in `Resources`) cannot use `!Sub`. They are plain strings. Use `&{VariableName}` directly in the key name — no quotes needed around `&{}` but they are allowed.

### Where ForEach Can Be Used

`Fn::ForEach` can appear in these sections of a template:

```
Resources   ✅  most common use — create multiple resources
Outputs     ✅  generate multiple outputs dynamically
```

It **cannot** be used inside individual resource property values directly. It operates at the resource block level, not the property value level.

### Realistic Examples

**Example 1 — Create S3 buckets for multiple environments**

```yaml
Transform: AWS::LanguageExtensions

Resources:
  'Fn::ForEach::CreateBuckets':
    - Env
    - [dev, staging, prod]
    - 'S3Bucket&{Env}':
        Type: AWS::S3::Bucket
        Properties:
          BucketName: !Sub 'my-app-&{Env}-bucket'
          Tags:
            - Key: Environment
              Value: !Sub '&{Env}'
```

This creates three buckets: `S3Bucketdev`, `S3Bucketstaging`, `S3Bucketprod`.

**Example 2 — Create security group rules for multiple ports**

```yaml
Transform: AWS::LanguageExtensions

Resources:
  MySecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Web server SG
      VpcId: !Ref MyVPC

  'Fn::ForEach::SGRules':
    - Port
    - ["80", "443", "8080"]
    - 'SGIngress&{Port}':
        Type: AWS::EC2::SecurityGroupIngress
        Properties:
          GroupId: !Ref MySecurityGroup
          IpProtocol: tcp
          FromPort: !Sub '&{Port}'
          ToPort: !Sub '&{Port}'
          CidrIp: 0.0.0.0/0
```

**Example 3 — Dynamic count from Mappings (this project)**

```yaml
Mappings:
  Config:
    dev:
      Count: ["1"]
    prod:
      Count: ["1", "2", "3"]

Resources:
  'Fn::ForEach::CreateInstances':
    - Instance
    - !FindInMap [Config, !Ref EnvType, Count]
    - 'EC2Instance&{Instance}':
        Type: AWS::EC2::Instance
        Properties:
          InstanceType: t3.micro
          ImageId: !Ref AMIId
          Tags:
            - Key: Name
              Value: !Sub 'instance-&{Instance}'
```

### ForEach with Outputs

```yaml
Outputs:
  'Fn::ForEach::InstanceOutputs':
    - Instance
    - ["1", "2"]
    - 'InstanceId&{Instance}':
        Description: !Sub 'Instance &{Instance} ID'
        Value: !GetAtt
          - !Sub 'EC2Instance&{Instance}'
          - InstanceId
```

---

## Fn::Length — Deep Dive

### What it does

`Fn::Length` returns the number of items in a list. It is part of the Language Extensions transform and requires `Transform: AWS::LanguageExtensions` to be declared.

### Syntax

```yaml
!Length
  - [item1, item2, item3]    # returns 3
```

Or using the full function form:

```yaml
Fn::Length:
  - [item1, item2, item3]
```

### Practical use cases

**Use case 1 — Show instance count in Outputs**

```yaml
Outputs:
  TotalInstances:
    Value:
      !Length
        - !FindInMap [InstanceTypeMap, !Ref EnvType, InstanceCount]
```

This is cleaner than the `!If [IsProd, "2", "1"]` approach used in Task 2 — it reads directly from the list rather than duplicating the count in a condition.

**Use case 2 — Validate list size in a condition**

```yaml
Conditions:
  HasMultipleInstances:
    !Not
      - !Equals
          - !Length
              - !FindInMap [InstanceTypeMap, !Ref EnvType, InstanceCount]
          - 1
```

**Use case 3 — Use count as a resource property value**

```yaml
Resources:
  MyASG:
    Type: AWS::AutoScaling::AutoScalingGroup
    Properties:
      MaxSize:
        !Length
          - !FindInMap [Config, !Ref EnvType, ServerList]
      MinSize: "1"
```

### Fn::Length vs hardcoded counts

| Approach | Maintainability |
|---|---|
| `!If [IsProd, "2", "1"]` | Must update the condition every time the list changes |
| `!Length [!FindInMap [...]]` | Automatically reflects the actual list size — single source of truth |

The `Fn::Length` approach is always preferred when the count corresponds to a list that already exists in the template.

### Important limitations

- `Fn::Length` only works on **lists** — passing a string or object causes a deployment error.
- The list must be resolvable at deploy time — it cannot be a dynamic value that depends on runtime resource attributes.
- Requires `Transform: AWS::LanguageExtensions` at the template level.

---

### FindInMap with Default Value (Language Extensions enhancement)

The standard `!FindInMap` fails if the key does not exist. The Language Extensions version adds an optional fourth argument for a default value:

```yaml
# Standard FindInMap — fails if key missing
!FindInMap [MapName, Key1, Key2]

# Language Extensions FindInMap — returns default if key missing
!FindInMap
  - MapName
  - !Ref EnvType
  - InstanceType
  - DefaultValue: t3.micro    # returned if the key path doesn't exist
```

This is useful when you have a large mapping but only want to define overrides for certain environments.

---

## Bugs and Gotchas

| Issue | Cause | Fix |
|---|---|---|
| `Transform: AWS::LanguageExtensions missing` | Fn::ForEach not recognised | Add `Transform: AWS::LanguageExtensions` at template level |
| `InsufficientCapabilities` | Transform requires explicit acknowledgement | Add `--capabilities CAPABILITY_AUTO_EXPAND` |
| Resource logical ID uses `${}` instead of `&{}` | Wrong substitution syntax for loop variable in IDs | Use `&{VariableName}` in resource keys |
| `python3.14 runtime not found` | 3.14 is not a released Lambda runtime | Use `python3.12` or `python3.11` |
| Lambda returns 502 Bad Gateway | `body` field in response is a dict not a string | Wrap body in `json.dumps()` |
| API GW returns 403 on invoke | Missing `AWS::Lambda::Permission` resource | Add `LambdaPermissionForApiGateway` resource |
| `Deployment` resource fails | Deployed before `MyApiMethod` is ready | Add `DependsOn: MyApiMethod` to Deployment |
| `Fn::Length` on a string | Passing non-list to Length | Ensure the value is a list `[]` |

---

## Key Concepts Learned

**AWS_PROXY integration** — API Gateway passes the raw HTTP request to Lambda as an event dict. Lambda owns the full response including status code, headers, and body. No mapping templates needed. The body in the Lambda response must always be a JSON string, not a dict.

**Lambda Permission resource** — a separate `AWS::Lambda::Permission` resource is required to allow API Gateway to invoke a Lambda function. The IAM role on the Lambda controls what the function *can do*. The Lambda Permission controls who *can call* the function. Both are needed.

**Transform: AWS::LanguageExtensions** — a pre-processor that runs before CloudFormation evaluates the template. It adds `Fn::ForEach`, `Fn::Length`, `Fn::ToJsonString`, and enhanced `FindInMap`. Must be declared at template level. Requires `CAPABILITY_AUTO_EXPAND`.

**Fn::ForEach** — iterates over a list and replicates a template block for each item. Uses `&{Variable}` for substitution in resource logical IDs and property values. The list can be hardcoded, come from a Mapping, or come from a Parameter. Eliminates duplicated resource blocks when count varies by environment.

**Fn::Length** — returns the count of items in a list. Keeps output values in sync with the actual list rather than requiring a separate hardcoded count.

**Single source of truth with Mappings** — by storing both the instance type and instance count list in the same Mappings block, a single `!FindInMap` call drives both the `ForEach` iteration and the `InstanceType` property. Changing the instance count means only updating the Mappings list — nothing else in the template needs to change.

**IMDSv2 and SSM AMI lookup** — using `AWS::SSM::Parameter::Value<AWS::EC2::Image::Id>` for the AMI ID means CloudFormation automatically resolves the latest AMI at deploy time. No need to hardcode region-specific AMI IDs or update the template when a new AMI is released.
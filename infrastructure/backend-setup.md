# Backend Setup: Lambda + API Gateway

## Overview

This guide sets up a serverless backend using AWS Lambda and API Gateway for handling any dynamic features (progress tracking, quiz submissions, notes, etc.).

## Architecture

```
API Gateway → Lambda Function → DynamoDB (optional)
     ↑
     └─ CloudFront (CORS enabled)
```

## Cost Analysis: Cheapest AWS Backend Service

### Option Comparison

| Service | Setup | Monthly Cost | Best For |
|---------|-------|--------------|----------|
| **Lambda + API Gateway** | ~30 min | <$1 | Dynamic features, perfect for study guide |
| EC2 (t3.micro) | ~1 hr | $5-10 | Always-on services |
| Lightsail | ~30 min | $3.50+ | Simple apps, managed |
| DynamoDB | ~15 min | <$1 | NoSQL database |
| RDS Micro | N/A | $15+ | SQL database |

**Selected: Lambda + API Gateway** (virtually free for this use case)

## Prerequisites

- AWS CLI configured
- IAM user with Lambda and API Gateway permissions
- Node.js 18+ installed locally

## Setup Steps

### 1. Create Lambda Execution Role

```bash
# Create trust policy document
cat > lambda-trust-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "lambda.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

# Create role
aws iam create-role \
  --role-name aif-c01-lambda-role \
  --assume-role-policy-document file://lambda-trust-policy.json

# Attach basic execution policy
aws iam attach-role-policy \
  --role-name aif-c01-lambda-role \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
```

### 2. Create Lambda Function

Create `lambda/index.js`:

```javascript
exports.handler = async (event) => {
  console.log('Received event:', JSON.stringify(event));

  const response = {
    statusCode: 200,
    headers: {
      'Content-Type': 'application/json',
      'Access-Control-Allow-Origin': '*',
      'Access-Control-Allow-Methods': 'GET, POST, OPTIONS',
      'Access-Control-Allow-Headers': 'Content-Type'
    },
    body: JSON.stringify({
      message: 'AIF-C01 Study Guide API',
      timestamp: new Date().toISOString(),
      path: event.path,
      method: event.httpMethod
    })
  };

  return response;
};
```

Package and deploy:

```bash
cd lambda
zip -r function.zip index.js

aws lambda create-function \
  --function-name aif-c01-api \
  --runtime nodejs18.x \
  --role arn:aws:iam::YOUR_ACCOUNT_ID:role/aif-c01-lambda-role \
  --handler index.handler \
  --zip-file fileb://function.zip

cd ..
```

### 3. Create API Gateway

```bash
# Create REST API
API_ID=$(aws apigateway create-rest-api \
  --name aif-c01-api \
  --description "Study guide backend API" \
  --query 'id' \
  --output text)

echo "API ID: $API_ID"

# Get root resource ID
ROOT_ID=$(aws apigateway get-resources \
  --rest-api-id $API_ID \
  --query 'items[0].id' \
  --output text)

# Create resource: /api
API_RESOURCE_ID=$(aws apigateway create-resource \
  --rest-api-id $API_ID \
  --parent-id $ROOT_ID \
  --path-part api \
  --query 'id' \
  --output text)

# Create resource: /api/{proxy+} for catch-all
PROXY_ID=$(aws apigateway create-resource \
  --rest-api-id $API_ID \
  --parent-id $API_RESOURCE_ID \
  --path-part '{proxy+}' \
  --query 'id' \
  --output text)
```

### 4. Create API Methods

```bash
# Create POST method
aws apigateway put-method \
  --rest-api-id $API_ID \
  --resource-id $PROXY_ID \
  --http-method ANY \
  --authorization-type NONE

# Integrate with Lambda
aws apigateway put-integration \
  --rest-api-id $API_ID \
  --resource-id $PROXY_ID \
  --http-method ANY \
  --type AWS_PROXY \
  --integration-http-method POST \
  --uri arn:aws:apigateway:us-east-1:lambda:path/2015-03-31/functions/arn:aws:lambda:us-east-1:YOUR_ACCOUNT_ID:function:aif-c01-api/invocations

# Enable CORS on root resource
aws apigateway put-method \
  --rest-api-id $API_ID \
  --resource-id $ROOT_ID \
  --http-method OPTIONS \
  --authorization-type NONE

aws apigateway put-integration \
  --rest-api-id $API_ID \
  --resource-id $ROOT_ID \
  --http-method OPTIONS \
  --type MOCK \
  --request-templates '{"application/json": "{\"statusCode\": 200}"}'
```

### 5. Grant API Gateway Permission to Invoke Lambda

```bash
aws lambda add-permission \
  --function-name aif-c01-api \
  --statement-id ApiGatewayInvoke \
  --action lambda:InvokeFunction \
  --principal apigateway.amazonaws.com \
  --source-arn "arn:aws:execute-api:us-east-1:YOUR_ACCOUNT_ID:$API_ID/*"
```

### 6. Deploy API

```bash
# Create stage
DEPLOY_ID=$(aws apigateway create-deployment \
  --rest-api-id $API_ID \
  --stage-name prod \
  --query 'id' \
  --output text)

echo "Deployment ID: $DEPLOY_ID"

# Get invoke URL
API_ENDPOINT="https://$API_ID.execute-api.us-east-1.amazonaws.com/prod"
echo "API Endpoint: $API_ENDPOINT"
```

### 7. Test API

```bash
# Test endpoint
curl -X GET "https://$API_ID.execute-api.us-east-1.amazonaws.com/prod/api/health"

# Should return:
# {
#   "message": "AIF-C01 Study Guide API",
#   "timestamp": "2026-05-07T...",
#   "path": "/api/health",
#   "method": "GET"
# }
```

## Optional: DynamoDB for Persistent Storage

If you need to track user progress or store quiz submissions:

```bash
# Create DynamoDB table
aws dynamodb create-table \
  --table-name aif-c01-users \
  --attribute-definitions \
    AttributeName=userId,AttributeType=S \
    AttributeName=timestamp,AttributeType=N \
  --key-schema \
    AttributeName=userId,KeyType=HASH \
    AttributeName=timestamp,KeyType=RANGE \
  --billing-mode PAY_PER_REQUEST

# Add policy to Lambda role for DynamoDB access
aws iam put-role-policy \
  --role-name aif-c01-lambda-role \
  --policy-name DynamoDBAccess \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Action": [
          "dynamodb:PutItem",
          "dynamodb:GetItem",
          "dynamodb:Query"
        ],
        "Resource": "arn:aws:dynamodb:us-east-1:YOUR_ACCOUNT_ID:table/aif-c01-users"
      }
    ]
  }'
```

## Cost Estimation

- Lambda: Free tier covers millions of requests; pay $0.20 per million requests
- API Gateway: $3.50/million requests
- DynamoDB: On-demand = $1.25 per million writes, $0.25 per million reads
- **Estimated Monthly Cost: <$1**

## Monitoring

```bash
# View Lambda logs
aws logs tail /aws/lambda/aif-c01-api --follow

# View API metrics
aws cloudwatch get-metric-statistics \
  --namespace AWS/ApiGateway \
  --metric-name Count \
  --dimensions Name=ApiName,Value=aif-c01-api \
  --start-time 2026-05-01T00:00:00Z \
  --end-time 2026-05-07T23:59:59Z \
  --period 86400 \
  --statistics Sum
```

## Next Steps

1. Implement specific endpoints for progress tracking
2. Add DynamoDB integration for persistent storage
3. Implement authentication (optional)
4. Set up automated deployments via GitHub Actions

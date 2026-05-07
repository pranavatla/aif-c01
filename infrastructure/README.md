# Infrastructure Setup Guide

Complete AWS deployment documentation for the AIF-C01 Study Guide.

## Quick Start

Follow these steps in order:

1. **S3 Setup** (`s3-setup.md`)
   - Create S3 bucket for static hosting
   - Enable website hosting
   - Configure bucket policies
   - Upload content

2. **CloudFront Setup** (`cloudfront-setup.md`)
   - Create CloudFront distribution
   - Configure custom domain (day1.aif.atla.in)
   - Set up HTTPS/SSL
   - Configure DNS records

3. **Backend Setup** (`backend-setup.md`)
   - Create Lambda function
   - Set up API Gateway
   - Optional: DynamoDB integration
   - Test endpoints

## Architecture Overview

```
┌─────────────────────────────────────────┐
│        User Browser                     │
│   visits day1.aif.atla.in               │
└─────────────┬───────────────────────────┘
              │ HTTPS
              ↓
┌─────────────────────────────────────────┐
│        Route 53 (DNS)                   │
│   day1.aif.atla.in → CloudFront         │
└─────────────┬───────────────────────────┘
              │
              ↓
┌─────────────────────────────────────────┐
│      CloudFront (CDN)                   │
│   Caches + distributes content          │
└─────────────┬───────────────────────────┘
              │
    ┌─────────┴─────────┐
    ↓                   ↓
┌──────────┐      ┌──────────────┐
│ S3       │      │ API Gateway  │
│ Static   │      │  + Lambda    │
│ Content  │      │ (Optional)   │
└──────────┘      └──────────────┘
                        │
                        ↓
                  ┌──────────────┐
                  │  DynamoDB    │
                  │  (Optional)  │
                  └──────────────┘
```

## Cost Summary

| Component | Monthly Cost |
|-----------|--------------|
| S3 Storage | <$1 |
| CloudFront Data Transfer | <$2 |
| Lambda Requests | <$0.50 |
| API Gateway | <$1 |
| DynamoDB (optional) | Free tier |
| **Total** | **<$5** |

## Deployment Scripts

All scripts are bash-based and require AWS CLI v2.

### One-Command Deployment

```bash
# Deploy everything in sequence
./deploy.sh

# Or deploy specific components
./scripts/deploy-s3.sh
./scripts/deploy-cloudfront.sh
./scripts/deploy-backend.sh
```

## Environment Variables

Create `.env` file in this directory:

```bash
AWS_ACCOUNT_ID=YOUR_ACCOUNT_ID
AWS_REGION=us-east-1
S3_BUCKET=aif-c01-static
CLOUDFRONT_DOMAIN=day1.aif.atla.in
LAMBDA_FUNCTION=aif-c01-api
API_GATEWAY_ID=YOUR_API_ID
```

## Troubleshooting

### S3 Access Denied
- Check bucket policy allows public read
- Verify IAM user has s3:* permissions

### CloudFront 403 Forbidden
- Ensure S3 bucket policy is public
- Check origin domain is correct
- Clear CloudFront cache

### API Gateway CORS Issues
- Verify CORS headers in Lambda response
- Check API Gateway CORS configuration
- Test with curl -H "Origin: ..." 

### DNS Not Resolving
- Wait 5-15 minutes for Route 53 propagation
- Verify CNAME record points to CloudFront domain
- Check Route 53 hosted zone configuration

## Security Considerations

1. **S3 Bucket**: Public read-only; no sensitive data
2. **CloudFront**: Use HTTPS only; enable compression
3. **Lambda**: No database credentials in environment; use IAM roles
4. **API Gateway**: Enable CloudWatch logging for monitoring
5. **DynamoDB**: Use on-demand pricing; delete old data regularly

## Monitoring

CloudWatch dashboards for tracking:
- CloudFront cache hit ratio
- Lambda execution duration
- API Gateway request count
- Error rates

View logs:
```bash
# CloudFront logs
aws cloudfront list-distributions

# Lambda logs
aws logs tail /aws/lambda/aif-c01-api --follow

# API Gateway logs
aws logs tail /aws/apigateway/aif-c01-api --follow
```

## Next Steps

1. Deploy infrastructure
2. Test all endpoints
3. Set up monitoring alerts
4. Document any custom configurations
5. Plan Days 2-4 content migration

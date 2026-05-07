# S3 Bucket Setup for Static Hosting

## Overview

This document outlines the steps to set up an S3 bucket for hosting the static study guide.

## Prerequisites

- AWS account with access to S3
- AWS CLI installed and configured
- Bucket name: `aif-c01-static` (adjust as needed)

## Steps

### 1. Create S3 Bucket

```bash
aws s3 mb s3://aif-c01-static --region us-east-1
```

### 2. Enable Static Website Hosting

```bash
aws s3 website s3://aif-c01-static/ \
  --index-document index.html \
  --error-document index.html
```

### 3. Create Bucket Policy for Public Read Access

Create a file `bucket-policy.json`:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "PublicReadGetObject",
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::aif-c01-static/*"
    }
  ]
}
```

Apply the policy:

```bash
aws s3api put-bucket-policy \
  --bucket aif-c01-static \
  --policy file://bucket-policy.json
```

### 4. Block Public Access Settings

Ensure "Block Public Access" is disabled (since we're making it public):

```bash
aws s3api put-public-access-block \
  --bucket aif-c01-static \
  --public-access-block-configuration \
  "BlockPublicAcls=false,IgnorePublicAcls=false,BlockPublicPolicy=false,RestrictPublicBuckets=false"
```

### 5. Enable Versioning (Recommended)

```bash
aws s3api put-bucket-versioning \
  --bucket aif-c01-static \
  --versioning-configuration Status=Enabled
```

### 6. Upload Content

```bash
# Upload day 1 materials
aws s3 sync day1/ s3://aif-c01-static/day1/ \
  --cache-control "public, max-age=3600"

# Upload day 2-4 (once available)
aws s3 sync day2/ s3://aif-c01-static/day2/ \
  --cache-control "public, max-age=3600"
```

### 7. Verify Upload

```bash
# List bucket contents
aws s3 ls s3://aif-c01-static/ --recursive

# Test access
curl http://aif-c01-static.s3-website-us-east-1.amazonaws.com/day1/index.html | head -20
```

## S3 Website Endpoint

- Website endpoint: `http://aif-c01-static.s3-website-us-east-1.amazonaws.com`
- REST endpoint: `http://aif-c01-static.s3.amazonaws.com`

(Note: Use CloudFront for production to enable HTTPS and custom domain)

## Deployment Script

Save as `deploy.sh`:

```bash
#!/bin/bash
set -e

BUCKET="aif-c01-static"

echo "Deploying to S3..."

# Upload all day directories
for day in day1 day2 day3 day4; do
  if [ -d "$day" ]; then
    echo "Uploading $day..."
    aws s3 sync "$day/" "s3://$BUCKET/$day/" \
      --delete \
      --cache-control "public, max-age=3600"
  fi
done

# Invalidate CloudFront cache (if using CloudFront)
# aws cloudfront create-invalidation --distribution-id YOUR_DISTRIBUTION_ID --paths "/*"

echo "Deployment complete!"
```

Make executable:
```bash
chmod +x deploy.sh
./deploy.sh
```

## Cost Estimation

- Storage: ~1-5MB per day ($0.023/GB/month) = minimal cost
- Data Transfer: CloudFront handles efficiently
- PUT/GET requests: Free tier covers most usage
- **Estimated Monthly Cost: <$1**

## Next Steps

1. Set up CloudFront distribution (see `cloudfront-setup.md`)
2. Configure custom domain pointing to CloudFront
3. Enable HTTPS with ACM certificate

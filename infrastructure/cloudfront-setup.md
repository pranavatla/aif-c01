# CloudFront Distribution Setup

## Overview

CloudFront provides CDN distribution, HTTPS support, and custom domain routing for the study guide.

## Prerequisites

- S3 bucket created and configured (see `s3-setup.md`)
- Route 53 domain management for atla.in
- AWS Certificate Manager (ACM) certificate (optional, can use default)

## Steps

### 1. Create CloudFront Distribution

Using AWS Console or CLI:

```bash
# Create distribution pointing to S3 bucket
aws cloudfront create-distribution \
  --origin-domain-name aif-c01-static.s3.amazonaws.com \
  --default-root-object index.html \
  --enabled
```

Or use this distribution config file `cloudfront-config.json`:

```json
{
  "CallerReference": "aif-c01-distribution",
  "Origins": {
    "Quantity": 1,
    "Items": [
      {
        "Id": "S3Origin",
        "DomainName": "aif-c01-static.s3.amazonaws.com",
        "S3OriginConfig": {
          "OriginAccessIdentity": ""
        }
      }
    ]
  },
  "DefaultCacheBehavior": {
    "AllowedMethods": {
      "Quantity": 7,
      "Items": ["GET", "HEAD", "OPTIONS", "PUT", "POST", "PATCH", "DELETE"]
    },
    "CachedMethods": {
      "Quantity": 2,
      "Items": ["GET", "HEAD"]
    },
    "TargetOriginId": "S3Origin",
    "ViewerProtocolPolicy": "redirect-to-https",
    "TrustedSigners": {
      "Enabled": false,
      "Quantity": 0
    },
    "ForwardedValues": {
      "QueryString": false,
      "Cookies": {
        "Forward": "none"
      }
    },
    "MinTTL": 0,
    "DefaultTTL": 86400,
    "MaxTTL": 31536000
  },
  "Comment": "CDN for AWS AIF-C01 Study Guide",
  "Enabled": true
}
```

Create distribution:

```bash
aws cloudfront create-distribution-with-config \
  --distribution-config file://cloudfront-config.json
```

### 2. Add Custom Domain (day1.aif.atla.in)

After distribution is created, add alternative domain names:

```bash
# Get distribution ID
DIST_ID=$(aws cloudfront list-distributions --query 'DistributionList.Items[0].Id' --output text)

# Update distribution with custom domain
aws cloudfront update-distribution \
  --id $DIST_ID \
  --distribution-config file://cloudfront-config-updated.json
```

In the distribution config, add:

```json
"Aliases": {
  "Quantity": 1,
  "Items": ["day1.aif.atla.in"]
}
```

### 3. Configure SSL/TLS Certificate

For HTTPS support with custom domain:

```bash
# Request ACM certificate for day1.aif.atla.in
aws acm request-certificate \
  --domain-name day1.aif.atla.in \
  --validation-method DNS \
  --region us-east-1
```

Add the certificate ARN to CloudFront distribution:

```json
"ViewerCertificate": {
  "ACMCertificateArn": "arn:aws:acm:us-east-1:YOUR_ACCOUNT_ID:certificate/CERTIFICATE_ID",
  "SSLSupportMethod": "sni-only",
  "MinimumProtocolVersion": "TLSv1.2_2021",
  "Certificate": "arn:aws:acm:us-east-1:YOUR_ACCOUNT_ID:certificate/CERTIFICATE_ID",
  "CertificateSource": "acm"
}
```

### 4. Create Route 53 DNS Record

In Route 53 for atla.in domain:

```bash
# Create CNAME record for day1 subdomain
aws route53 change-resource-record-sets \
  --hosted-zone-id YOUR_HOSTED_ZONE_ID \
  --change-batch '{
    "Changes": [
      {
        "Action": "CREATE",
        "ResourceRecordSet": {
          "Name": "day1.aif.atla.in",
          "Type": "CNAME",
          "TTL": 300,
          "ResourceRecords": [
            {
              "Value": "d123abc456.cloudfront.net"
            }
          ]
        }
      }
    ]
  }'
```

Or use AWS Console:
- Go to Route 53 → Hosted Zones → atla.in
- Create new record: `day1.aif.atla.in` → CNAME → CloudFront domain name

### 5. Test CloudFront Distribution

```bash
# Test through CloudFront domain
curl https://d123abc456.cloudfront.net/day1/index.html | head -20

# Test through custom domain (after DNS propagates, ~15 mins)
curl https://day1.aif.atla.in/day1/index.html | head -20
```

### 6. Configure Cache Behaviors

For optimal performance, add behaviors for different content types:

**For HTML files** (shorter cache):
```json
{
  "PathPattern": "day*/*.html",
  "ViewerProtocolPolicy": "redirect-to-https",
  "DefaultTTL": 3600,
  "MaxTTL": 86400,
  "Compress": true
}
```

**For static assets** (longer cache):
```json
{
  "PathPattern": "day*/*.{js,css,png,jpg,gif,svg,woff}",
  "ViewerProtocolPolicy": "redirect-to-https",
  "DefaultTTL": 31536000,
  "MaxTTL": 31536000,
  "Compress": true
}
```

### 7. Enable Compression

Reduce bandwidth costs by enabling gzip compression:

```json
"Compress": true,
"DefaultCacheBehavior": {
  "Compress": true
}
```

### 8. Monitor Performance

```bash
# Get CloudFront metrics
aws cloudwatch get-metric-statistics \
  --namespace AWS/CloudFront \
  --metric-name BytesDownloaded \
  --dimensions Name=DistributionId,Value=$DIST_ID \
  --start-time 2026-05-01T00:00:00Z \
  --end-time 2026-05-07T23:59:59Z \
  --period 86400 \
  --statistics Sum
```

## Cost Estimation

- Data transfer: $0.085/GB (US)
- HTTP requests: $0.0075/10,000
- HTTPS requests: $0.0100/10,000
- **Estimated Monthly Cost: <$2 (based on 100GB/month)**

## Troubleshooting

**404 Errors**: Ensure index.html exists in S3 and S3 bucket policy allows public read

**Slow Performance**: Check CloudFront cache statistics; ensure TTL values are appropriate

**SSL Certificate Issues**: Verify certificate validation in Route 53; may take 15 minutes

**DNS Not Resolving**: Verify Route 53 record is correctly pointing to CloudFront domain

## Next Steps

1. Set up backend API (see `backend-setup.md`)
2. Monitor CloudFront metrics
3. Implement cache invalidation strategy

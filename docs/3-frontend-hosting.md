---
sidebar_position: 3
---

# Frontend Static Hosting

Static website hosting for the frontend application in AWS is configured using **Amazon S3** and **Amazon CloudFront**.  

Access between CloudFront and S3 is secured through **Origin Access Control (OAC)**, ensuring the bucket is not publicly accessible while still allowing CloudFront to serve content globally.

---
### Overview

**Components:**
- **Amazon S3** — Hosts the static frontend assets (HTML, CSS, JS).
- **Origin Access Control (OAC)** — Restricts S3 access to CloudFront only.
- **Amazon CloudFront** — Provides content delivery, caching, and HTTPS support.

---
### Remote State Configuration
###### <i>modules/frontend/main.tf</i>
```
data "terraform_remote_state" "s3" {
  backend = "s3"

  config = {
    bucket = var.db_remote_state_bucket
    key    = var.db_remote_state_key
    region = var.aws_region
  }
}
```
<sub><i>This block retrieves outputs from the previously configured remote Terraform state stored in S3 bucket `mc-remote-state`, using the specified configuration parameters</i></sub>

---
## Infrastructure Provisioning
### S3 Bucket 

The S3 bucket hosts static frontend assets and enforces strict access control through policies and public access blocking.

```
resource "aws_s3_bucket" "frontend" {
  bucket = "${var.project_name}-static-25"
  region = var.aws_region

  tags = {
    Name        = "${var.project_name}-static-25"
  }
}

resource "aws_s3_bucket_public_access_block" "block_public" {
  bucket = aws_s3_bucket.frontend.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

resource "aws_s3_bucket_website_configuration" "frontend" {
  bucket = aws_s3_bucket.frontend.id

  index_document {
    suffix = "index.html"
  }

  error_document {
    key = "index.html"
  }
}
```
- Creates a dedicated, uniquely named S3 bucket for static website assets.
- Ensures the bucket and its objects are completely private.
- Enables static website hosting on S3 with `index.html` as both index and error document.

### Origin Access Control (OAC) 

The OAC configuration allows CloudFront to access the S3 bucket securely using the CloudFront service principal and OAC signing.

```
data "aws_iam_policy_document" "origin_bucket_policy" {
  statement {
    sid    = "AllowCloudFrontServicePrincipalReadWrite"
    effect = "Allow"

    principals {
      type        = "Service"
      identifiers = ["cloudfront.amazonaws.com"]
    }

    actions = [
      "s3:GetObject",
      "s3:PutObject",
    ]

    resources = [
      "${aws_s3_bucket.frontend.arn}/*",
    ]

    condition {
      test     = "StringEquals"
      variable = "AWS:SourceArn"
      values   = [aws_cloudfront_distribution.s3_distribution.arn]
    }
  }
}

resource "aws_s3_bucket_policy" "frontend" {
  bucket = aws_s3_bucket.frontend.id
  policy = data.aws_iam_policy_document.origin_bucket_policy.json
}
```
- Grants CloudFront permission to read objects directly from S3.
- Access is scoped by the CloudFront distribution ARN for least-privilege security.
- Uses service principal `cloudfront.amazonaws.com` and SigV4 signing via OAC.
- Ensures CloudFront can fetch objects while public access remains blocked.

```
resource "aws_cloudfront_origin_access_control" "default" {
  name                              = "default-oac"
  origin_access_control_origin_type = "s3"
  signing_behavior                  = "always"
  signing_protocol                  = "sigv4"
}
```
- Configures a CloudFront OAC for secure communication with S3.
- `signing_behavior = "always"` enforces consistent SigV4 request signing.

### CloudFront Distribution

The CloudFront distribution acts as the global CDN for the S3 bucket, providing secure HTTPS access and caching optimization.

```
locals {
  s3_origin_id = "S3Origin"
}

resource "aws_cloudfront_distribution" "s3_distribution" {
  origin {
    domain_name              = aws_s3_bucket.frontend.bucket_regional_domain_name
    origin_access_control_id = aws_cloudfront_origin_access_control.default.id
    origin_id                = local.s3_origin_id
  }

  enabled             = true
  is_ipv6_enabled     = true
  default_root_object = "index.html"

  default_cache_behavior {
    allowed_methods  = ["DELETE", "GET", "HEAD", "OPTIONS", "PATCH", "POST", "PUT"]
    cached_methods   = ["GET", "HEAD"]
    target_origin_id = local.s3_origin_id

    forwarded_values {
      query_string = false

      cookies {
        forward = "none"
      }
    }
    viewer_protocol_policy = "redirect-to-https"
    min_ttl                = 0
    default_ttl            = 3600
    max_ttl                = 86400
  }

  restrictions {
    geo_restriction {
      locations        = []
      restriction_type = "none"
    }
  }

  tags = {
    Environment = "Dev"
  }

  viewer_certificate {
    cloudfront_default_certificate = true
  }

}

```

- **Origin Configuration:** Points to the S3 bucket’s regional domain with OAC-enabled access.
- **HTTPS Enforcement:** `viewer_protocol_policy = "redirect-to-https"` ensures secure requests.
- **Caching Strategy:**
    - Default TTL: 1 hour for balance between freshness and performance.
    - Max TTL: 24 hours for static assets.
    - Cache Methods: Only `GET` and `HEAD` cached to prevent invalidation issues.
- **Geo Restriction:** Disabled to allow global content delivery.
- **IPv6 Enabled:** Ensures compatibility with modern clients.
- **Viewer Certificate:** Uses CloudFront’s default SSL certificate for simplicity (custom certs can be added later).
- **WAF and Security Protections:** Not enabled in this configuration to simplify initial deployment; can be integrated later.

---

## References

- **Full Source:** [Github Link](https://github.com/deeowemez/minicommerce/blob/main/infra/modules/frontend/main.tf)
- **AWS Docs:**
    - Using CloudFront Origin Access Control (OAC)
    - Hosting a Static Website on Amazon S3

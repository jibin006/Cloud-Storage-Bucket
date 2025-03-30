# Challenge 1: Secure a Cloud Storage Bucket

## Objective
The goal of this challenge was to create a secure AWS S3 bucket using Terraform, ensuring that:
- The bucket is private and not publicly accessible.
- Only a specific user (`jib_iam`) has access, while another user (`jib02`) is denied.
- Encryption is enabled using server-side encryption with AWS KMS.
- Unauthorized access attempts are logged and can be verified.

## Success Criteria
- Unauthorized users (e.g., `jib02`) cannot access the bucket.
- Logs show failed access attempts.
- Encryption is enabled, and data is secure.

## Approach
I used Terraform to automate the creation and configuration of the S3 bucket. My script included:
1. **S3 Bucket Creation**: Defined a private S3 bucket with a unique name.
2. **Public Access Block**: Blocked all public access to the bucket.
3. **IAM Policy**: Allowed access only to the `jib_iam` user and denied `jib02`.
4. **KMS Encryption**: Enabled server-side encryption with a custom KMS key.
5. **Logging**: Configured a separate S3 bucket to log access attempts.

Here’s the final Terraform script that met all requirements:

```hcl
resource "aws_s3_bucket" "example" {
  bucket = "my-tf-example-bucketjib"
}

resource "aws_s3_bucket_ownership_controls" "example" {
  bucket = aws_s3_bucket.example.id
  rule {
    object_ownership = "BucketOwnerPreferred"
  }
}

resource "aws_s3_bucket_acl" "example" {
  depends_on = [aws_s3_bucket_ownership_controls.example]
  bucket = aws_s3_bucket.example.id
  acl    = "private"
}

resource "aws_s3_bucket_public_access_block" "example" {
  bucket                  = aws_s3_bucket.example.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

data "aws_iam_policy_document" "combined_bucket_policy" {
  statement {
    effect = "Allow"
    principals {
      type        = "AWS"
      identifiers = ["arn:aws:iam::816069160759:user/jib_iam"]
    }
    actions = ["s3:GetObject", "s3:ListBucket"]
    resources = [aws_s3_bucket.example.arn, "${aws_s3_bucket.example.arn}/*"]
  }
  statement {
    effect = "Deny"
    principals {
      type        = "AWS"
      identifiers = ["arn:aws:iam::816069160759:user/jib02"]
    }
    actions = ["s3:GetObject", "s3:ListBucket"]
    resources = [aws_s3_bucket.example.arn, "${aws_s3_bucket.example.arn}/*"]
  }
}

resource "aws_s3_bucket_policy" "bucket_policy" {
  bucket = aws_s3_bucket.example.id
  policy = data.aws_iam_policy_document.combined_bucket_policy.json
  depends_on = [aws_s3_bucket.example]
}

resource "aws_kms_key" "mykey" {
  description             = "This key is used to encrypt bucket objects"
  deletion_window_in_days = 10
}

resource "aws_s3_bucket_server_side_encryption_configuration" "example" {
  bucket = aws_s3_bucket.example.id
  rule {
    apply_server_side_encryption_by_default {
      kms_master_key_id = aws_kms_key.mykey.arn
      sse_algorithm     = "aws:kms"
    }
  }
}

resource "aws_s3_bucket" "log_bucket" {
  bucket = "my-tf-logs-bucketjib06"
}

resource "aws_s3_bucket_logging" "example" {
  bucket        = aws_s3_bucket.example.id
  target_bucket = aws_s3_bucket.log_bucket.id
  target_prefix = "logs/"
}
```

## Issues Faced and Resolutions

### Issue 1: Bucket Name Conflicts
**Problem**: When I tried to create the S3 buckets (`my-tf-example-bucket` and `my-tf-logs-bucket`), I got a `BucketAlreadyExists` error because the names were already taken by another AWS account.

**Resolution**: Since S3 bucket names must be globally unique, I added a unique suffix to the bucket names (e.g., `my-tf-example-bucketjib` and `my-tf-logs-bucketjib06`). This ensured the names were unique and avoided conflicts.

**How I Fixed It**: Manually modified the `bucket` attribute in the `aws_s3_bucket` resources:
```hcl
bucket = "my-tf-example-bucketjib"
bucket = "my-tf-logs-bucketjib06"
```

### Issue 2: Invalid Principal in Bucket Policy
**Problem**: I got a `MalformedPolicy: Invalid principal in policy` error when applying the bucket policy. I initially used the AWS account ID (`816069160759`) instead of a proper ARN.

**Resolution**: I updated the policy to use the full ARN of the specific user (`arn:aws:iam::816069160759:user/jib_iam`) instead of just the account ID.

**How I Fixed It**: Corrected the `identifiers` in the policy:
```hcl
identifiers = ["arn:aws:iam::816069160759:user/jib_iam"]
```

### Issue 3: Both Users Could Access the Bucket
**Problem**: Even after specifying `jib_iam` in the bucket policy, the other user (`jib02`) could still access the bucket. This was because the policy only allowed `jib_iam` but didn’t explicitly deny `jib02`, and `jib02` might have had access via other permissions.

**Resolution**: I added an explicit `Deny` statement for `jib02` in the bucket policy to ensure only `jib_iam` had access.

**How I Fixed It**: Combined both `Allow` and `Deny` statements into one policy document:
```hcl
data "aws_iam_policy_document" "combined_bucket_policy" {
  statement {
    effect = "Allow"
    principals {
      type        = "AWS"
      identifiers = ["arn:aws:iam::816069160759:user/jib_iam"]
    }
    actions = ["s3:GetObject", "s3:ListBucket"]
    resources = [aws_s3_bucket.example.arn, "${aws_s3_bucket.example.arn}/*"]
  }
  statement {
    effect = "Deny"
    principals {
      type        = "AWS"
      identifiers = ["arn:aws:iam::816069160759:user/jib02"]
    }
    actions = ["s3:GetObject", "s3:ListBucket"]
    resources = [aws_s3_bucket.example.arn, "${aws_s3_bucket.example.arn}/*"]
  }
}
```

### Issue 4: Bucket Policy Timing Issue
**Problem**: I encountered an error: `couldn't find resource` when applying the bucket policy, likely because Terraform tried to apply the policy before the bucket was fully created.

**Resolution**: I added a `depends_on` attribute to the `aws_s3_bucket_policy` resource to ensure the bucket was created first.

**How I Fixed It**: Added the dependency:
```hcl
resource "aws_s3_bucket_policy" "bucket_policy" {
  bucket = aws_s3_bucket.example.id
  policy = data.aws_iam_policy_document.combined_bucket_policy.json
  depends_on = [aws_s3_bucket.example]
}
```

## Final Outcome
After resolving these issues, the Terraform script successfully:
- Created a private S3 bucket (`my-tf-example-bucketjib`) with a unique name.
- Blocked all public access.
- Allowed only `jib_iam` to access the bucket while denying `jib02`.
- Enabled server-side encryption with AWS KMS.
- Configured logging to a separate bucket (`my-tf-logs-bucketjib06`).

**Testing Results**:
- `jib_iam` could access the bucket (success).
- `jib02` was denied access (success).
- Logs in the separate bucket showed failed access attempts by `jib02`.
- Encryption was verified via the AWS Console.
  <img width="710" alt="428391528-ae76dabf-e10d-4073-a06a-b2bff83986cf" src="https://github.com/user-attachments/assets/65f5dcb9-3265-43e1-b916-d1cd8e5caa3f" />


## Lessons Learned
- **Bucket Naming**: S3 bucket names need to be globally unique, so adding a suffix is a simple fix.
- **IAM Policies**: Be precise with ARNs and use explicit `Deny` rules to restrict unwanted access.
- **Dependencies**: Use `depends_on` in Terraform to avoid timing issues with resource creation.

## Resource
https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/s3_bucket#acl-1

This challenge taught me how to secure cloud storage effectively and troubleshoot common configuration errors in Terraform and AWS.

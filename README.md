# Challenge 1: Secure a Cloud Storage Bucket

## Overview
This challenge involved creating and configuring an AWS S3 bucket using Terraform. The goal was to apply best practices such as access control, encryption, logging, and bucket policies. I used Terraform for the completion of this challenge.

🛠 Task:

Create an S3 bucket (AWS), Cloud Storage bucket (GCP), or Blob Storage (Azure).

Disable public access and ensure it's private.

Set up IAM roles to allow access only to specific users/services.

Enable encryption (SSE-KMS in AWS, CMEK in GCP, Azure SSE).

Test unauthorized access using another account and check logs.

✅ Success Criteria:

Unauthorized users cannot access the bucket.

Logs show failed access attempts.

Encryption is enabled, and data is secure.

## Resources Created
- **S3 Bucket** (`aws_s3_bucket.example`)
- **Bucket Ownership Controls** (`aws_s3_bucket_ownership_controls.example`)
- **Bucket ACL** (`aws_s3_bucket_acl.example`)
- **Public Access Block** (`aws_s3_bucket_public_access_block.example`)
- **Bucket Policy** (`aws_s3_bucket_policy.bucket_policy`)
- **IAM Policy Documents** to manage access restrictions
- **KMS Key** for server-side encryption (`aws_kms_key.mykey`)
- **Bucket Encryption Configuration** (`aws_s3_bucket_server_side_encryption_configuration.example`)
- **S3 Logging Configuration** (`aws_s3_bucket_logging.example`)

## Issues Faced & Solutions

### ❌ Issue 1: "Couldn't find resource" error while applying S3 bucket policy
#### **Error Message:**
```sh
│ Error: waiting for S3 Bucket Policy (my-tf-example-bucketjib) create: couldn't find resource
```
#### **Cause:**
Terraform was attempting to apply the bucket policy before the S3 bucket was fully created, causing a race condition.

#### **Solution:**
Added an explicit dependency on the S3 bucket in the bucket policy resource:
```hcl
resource "aws_s3_bucket_policy" "bucket_policy" {
  bucket = aws_s3_bucket.example.id
  policy = data.aws_iam_policy_document.combined_bucket_policy.json

  depends_on = [aws_s3_bucket.example]
}
```

### ❌ Issue 2: Incorrect IAM Policy Document Formatting
#### **Cause:**
The IAM policy document was incorrectly structured, leading to syntax issues.

#### **Solution:**
- Ensured the policy had proper JSON syntax.
- Corrected the **Deny** and **Allow** statements within `data "aws_iam_policy_document"`.

### ❌ Issue 3: Misconfigured Encryption
#### **Cause:**
The S3 encryption was failing due to missing KMS key reference.

#### **Solution:**
- Created an AWS KMS key (`aws_kms_key.mykey`).
- Linked it properly in the S3 encryption configuration:
  ```hcl
  resource "aws_s3_bucket_server_side_encryption_configuration" "example" {
    bucket = aws_s3_bucket.example.id

    rule {
      apply_server_side_encryption_by_default {
        kms_master_key_id = aws_kms_key.mykey.arn
        sse_algorithm     = "aws:kms"
      }
    }
  }
  ```

## Steps to Apply Fixes
1. **Ensure Terraform configuration is correct:**
   ```sh
   terraform fmt
   terraform validate
   ```
2. **Refresh the state:**
   ```sh
   terraform apply -refresh=true
   ```
3. **Apply the updated configuration:**
   ```sh
   terraform apply
   ```

<img width="710" alt="image" src="https://github.com/user-attachments/assets/ae76dabf-e10d-4073-a06a-b2bff83986cf" />


## Resource
   https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/s3_bucket#acl-1

## Conclusion
This challenge demonstrated the importance of **Terraform dependency management**, **IAM policy structuring**, and **proper encryption configurations**. By adding explicit dependencies and fixing syntax errors, the deployment was successfully completed. 🚀

## 🔄 Workflow Overview

This pipeline is designed to securely execute operations across AWS accounts using cross-account role assumption and then upload results back to an S3 bucket in the source account.

---

## 🏗️ Architecture

* **Account A (Source Account)**

  * Hosts Jenkins server (EC2)
  * EC2 instance has an IAM role attached
  * Permissions:

    * Assume role into target accounts
    * Upload objects to S3 bucket

* **Account B (Target Account)**

  * Contains resources to be accessed (e.g., EC2)
  * Has IAM role (`Jenkins-CrossAccount-Admin`) trusted by Account A

---

## 🔁 Execution Flow

### 1. Input Parameters

* User provides:

  * `TARGET_ACCOUNT_ID`
  * `REGION` (e.g., `ap-south-1`)

---

### 2. Assume Role in Target Account

* Jenkins (running in Account A) uses its EC2 IAM role to assume a role in Account B:

```id="9b9g8x"
aws sts assume-role \
  --role-arn arn:aws:iam::<TARGET_ACCOUNT_ID>:role/<CROSS_ACCOUNT_ROLE> \
  --role-session-name jenkins-session
```

* Temporary credentials are exported:

  * `AWS_ACCESS_KEY_ID`
  * `AWS_SECRET_ACCESS_KEY`
  * `AWS_SESSION_TOKEN`

---

### 3. Execute Operations in Target Account

* Using assumed role credentials, the pipeline performs required actions in Account B:

  * Example:

    * Fetch EC2 details
    * Tag resources
    * Generate reports

---

### 4. Reset Credentials (Critical Step)

* After completing operations in Account B, temporary credentials are cleared:

```id="f8t6pa"
unset AWS_ACCESS_KEY_ID AWS_SECRET_ACCESS_KEY AWS_SESSION_TOKEN
```

* This ensures the pipeline reverts back to:

  * **Jenkins EC2 IAM role (Account A)**

---

### 5. Upload Results to S3 (Account A)

* Using the original EC2 role permissions, the pipeline uploads output files to S3:

```id="pl0z3g"
aws s3 cp <output-file> s3://<bucket-name>/<path>/
```

---

## 🔐 Security Considerations

* Uses **temporary STS credentials** (no hardcoded secrets)
* Ensures **least privilege access**
* Clears credentials after cross-account operations
* Prevents accidental use of target account credentials for S3 uploads

---

## ⚠️ Important Notes

* The `unset` step is mandatory to avoid:

  * Upload failures
  * Cross-account permission issues
* Ensure:

  * Trust relationship is configured correctly in target account
  * Jenkins EC2 role has `sts:AssumeRole` permission
  * S3 bucket policy allows uploads from Jenkins role

---

## 🧭 Summary

1. Start in Account A (Jenkins EC2 role)
2. Assume role into Account B
3. Perform required operations
4. Unset temporary credentials
5. Upload results to S3 in Account A

---

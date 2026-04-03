# Jenkins AWS EC2 Inventory Pipeline

This repository contains a Jenkins pipeline that collects EC2 instance inventory across multiple AWS accounts using cross-account role assumption and uploads the results to an S3 bucket.

---

## 🚀 Features

* 🔁 Cross-account access using IAM role
* 📊 EC2 inventory extraction (JSON → CSV)
* 📦 Uploads reports to S3
* ⚡ Parallel execution for multiple accounts
* 🌍 Region configurable
* 💾 Includes volume details (IDs and sizes)

---

## 🛠️ Parameters

| Parameter            | Description                     | Example                     |
| -------------------- | ------------------------------- | --------------------------- |
| `TARGET_ACCOUNT_IDS` | Comma-separated AWS account IDs | `123456789012,987654321098` |
| `REGION`             | AWS region                      | `ap-south-1`                |

---

## 🔐 Prerequisites

* Jenkins with AWS CLI installed
* IAM Role in target accounts:

  * Role Name: `Jenkins-CrossAccount-Admin`
  * Trust policy allowing Jenkins account
* S3 bucket:

  * `jenkins-log-files-s3-bucket`

---

## 📂 Output

CSV file uploaded to:

```
s3://jenkins-log-files-s3-bucket/ec2-inventory/<account-id>/
```

---

## 📄 Sample CSV Columns

* Name
* InstanceID
* InstanceState
* InstanceType
* AvailabilityZone
* PrivateIP
* PublicIP
* VolumeIDs
* VolumeSizes
* VPCID
* SubnetIDs

---

## ⚙️ How It Works

1. Jenkins assumes role in target account
2. Fetches EC2 instance details
3. Extracts metadata using `jq`
4. Converts to CSV
5. Uploads to S3

---

## ⚠️ Notes

* Ensure `jq` is installed on Jenkins agent
* Volume size fetching may increase execution time
* Proper IAM permissions are required for:

  * `sts:AssumeRole`
  * `ec2:DescribeInstances`
  * `ec2:DescribeVolumes`
  * `s3:PutObject`

---

## 👨‍💻 Author

Maintained for DevOps automation and cloud inventory tracking.

---

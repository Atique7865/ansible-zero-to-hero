# Enable MFA Delete on S3 Bucket Using Google Authenticator

MFA Delete adds an extra layer of protection to S3 versioned buckets by requiring a one-time password (OTP) before permanently deleting object versions or disabling versioning.

---

## Prerequisites

- AWS CLI installed and configured
- AWS root account credentials (MFA Delete can only be enabled by the **root account**)
- Google Authenticator app installed on your mobile device
- A **virtual MFA device** already registered to your AWS root account via Google Authenticator

> ⚠️ MFA Delete **cannot** be enabled using IAM user credentials — it requires the root account.

---

## Step 1: Set Up Google Authenticator as a Virtual MFA Device (if not already done)

1. Sign in to the [AWS Management Console](https://console.aws.amazon.com/) as **root user**
2. Go to **IAM** → **Security credentials** (under your account menu, top right)
3. Scroll to **Multi-factor authentication (MFA)** → click **Assign MFA device**
4. Choose **Authenticator app** → click **Next**
5. Open **Google Authenticator** on your phone → tap **+** → **Scan a QR code**
6. Scan the QR code shown on AWS
7. Enter **two consecutive OTP codes** from Google Authenticator to verify
8. Click **Add MFA** — your virtual MFA device is now registered

Note down the **MFA device ARN** (e.g., `arn:aws:iam::123456789012:mfa/root-account-mfa-device`) — you'll need it in Step 3.

---

## Step 2: Enable Versioning on the S3 Bucket

MFA Delete requires versioning to be enabled first.

```bash
aws s3api put-bucket-versioning \
  --bucket YOUR_BUCKET_NAME \
  --versioning-configuration Status=Enabled
```

Verify versioning is enabled:

```bash
aws s3api get-bucket-versioning --bucket YOUR_BUCKET_NAME
```

Expected output:
```json
{
    "Status": "Enabled"
}
```

---

## Step 3: Enable MFA Delete

You need:
- Your **MFA device ARN** (from Step 1)
- A **valid OTP code** from Google Authenticator (6-digit code)
- Root account AWS credentials set in your environment

### Set root credentials in environment variables

```bash
export AWS_ACCESS_KEY_ID=YOUR_ROOT_ACCESS_KEY_ID
export AWS_SECRET_ACCESS_KEY=YOUR_ROOT_SECRET_ACCESS_KEY
```

### Enable MFA Delete

```bash
aws s3api put-bucket-versioning \
  --bucket YOUR_BUCKET_NAME \
  --versioning-configuration Status=Enabled,MFADelete=Enabled \
  --mfa "arn:aws:iam::123456789012:mfa/root-account-mfa-device YOUR_OTP_CODE"
```

Replace:
| Placeholder | Value |
|---|---|
| `YOUR_BUCKET_NAME` | Your S3 bucket name |
| `arn:aws:iam::123456789012:mfa/root-account-mfa-device` | Your MFA device ARN |
| `YOUR_OTP_CODE` | Current 6-digit OTP from Google Authenticator |

> ⏱️ OTP codes expire in 30 seconds — run the command immediately after reading the code.

### Verify MFA Delete is enabled

```bash
aws s3api get-bucket-versioning --bucket YOUR_BUCKET_NAME
```

Expected output:
```json
{
    "Status": "Enabled",
    "MFADelete": "Enabled"
}
```

---

## Step 4: Test MFA Delete Behavior

With MFA Delete enabled, deleting an object version **without** MFA will be denied:

```bash
# This will FAIL without MFA token
aws s3api delete-object \
  --bucket YOUR_BUCKET_NAME \
  --key YOUR_OBJECT_KEY \
  --version-id YOUR_VERSION_ID
```

To permanently delete a version **with** MFA:

```bash
aws s3api delete-object \
  --bucket YOUR_BUCKET_NAME \
  --key YOUR_OBJECT_KEY \
  --version-id YOUR_VERSION_ID \
  --mfa "arn:aws:iam::123456789012:mfa/root-account-mfa-device YOUR_OTP_CODE"
```

---

## Step 5: Disable MFA Delete (when needed)

To disable MFA Delete, also requires MFA authentication:

```bash
aws s3api put-bucket-versioning \
  --bucket YOUR_BUCKET_NAME \
  --versioning-configuration Status=Enabled,MFADelete=Disabled \
  --mfa "arn:aws:iam::123456789012:mfa/root-account-mfa-device YOUR_OTP_CODE"
```

---

## Summary

| Step | Action |
|------|--------|
| 1 | Register Google Authenticator as virtual MFA on AWS root account |
| 2 | Enable versioning on the S3 bucket |
| 3 | Enable MFA Delete using root credentials + OTP |
| 4 | All version deletions now require MFA token |
| 5 | To disable, repeat with `MFADelete=Disabled` |

---

## Important Notes

- 🔐 Only the **root account** can enable/disable MFA Delete
- 📱 Always keep your Google Authenticator device safe — losing it requires AWS support to recover
- 🔄 OTP codes are time-based (TOTP) and valid for ~30 seconds
- 💰 MFA Delete does **not** incur additional AWS charges
- 🗂️ Bucket versioning **cannot** be suspended while MFA Delete is enabled

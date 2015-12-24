---
layout: post
title: AWS Key Management Service
author: Richard Hurt
tags: AWS KMS
---

![KMS Logo](https://d0.awsstatic.com/product-marketing/KMS/KMS_Benefit_Key.png "KMS Logo"){: .right}
I have a need to protect secrets at rest and AWS has thoughtfully provided a solution called
[Key Management Service (KMS)](https://aws.amazon.com/kms/).  Using this tool I can put encrypted
files almost anywhere and be safe in the knowledge that only I (or systems that I authorize) can
decrypt the contents.


## Steps to encrypt your data at rest

### 1) Create a new IAM Encryption Key

Even if you already have a master key it's probably a good idea to create a new key for every purpose.
So, if you're encrypting VPN credentials, create a key.  If you're encrypting Github credentials, create
a different key.  This keeps things nice and segregated so if there is a problem in the future with your
secrets, you can revoke it's key without affecting the other secrets.

### 2a) Encrypt your data (small amounts <= 4kb)

1. Get the Key ID from the above step.  This will look something like this: 3aaaaaaaa-fccc-4bbb-bccc-addddddddd
2. Encode your data using the following command (replacing "your_data_file" and "3aaaaaaaa-fccc-4bbb-bccc-addddddddd"
with data specific to your situation).  The output will be an encrypted version of your input data
encoded as a Base64 file, suitable for emailing or cut-n-pasting into documents.

```
      base64 your_data_file | aws kms encrypt --key-id 3aaaaaaaa-fccc-4bbb-bccc-addddddddd
      --output text --query CiphertextBlob --plaintext - > your_data_file.asc
```

_NOTE: The KMS system likes to deal with ASCII data and feeding it binary
data is generally considered a bad thing (at least I haven't figured out how to do it yet)._

### 2b) Encrypt your data (large amounts > 4kb)

1. Get the Key ID from the above step.  This will look something like this: 3aaaaaaaa-fccc-4bbb-bccc-addddddddd
2. Encode your data using the following command (replacing "your_data_file" and "3aaaaaaaa-fccc-4bbb-bccc-addddddddd"
with data specific to your situation).  The output will be an encrypted version of your input data
encoded as a Base64 file, suitable for emailing or cut-n-pasting into documents.

```
      base64 your_data_file | aws kms encrypt --key-id 3aaaaaaaa-fccc-4bbb-bccc-addddddddd
      --output text --query CiphertextBlob --plaintext - > your_data_file.asc
```

# 8. Pre-Signed URLs

## 8.0 Pain points
- to control from a central point pre-signed URLs
- to identify abuse of non authenticated requests


## 8.1 What is an S3 pre-signed URL?
A pre-signed URL is meant to be easily shared and available for a limited amount of time. Think about it like a file transfert link without the need of sharing an access key and a secret key that you share with an external user for a couple of days. This link would be available to GET (download) a specific file or PUT a file (upload) to a bucket.
After the admin specified expiration date, the link is not available anymore.


a pre-signed URL can be created to upload an object to a bucket for an amount of time (here it is configured for 168 hours, therefore 7 days):
```shell
mc share upload minio-101/bucket1/test.txt --expire 168h
URL: http://10.1.10.101:9000/bucket1/test.txt
Expire: 7 days 0 hours 0 minutes 0 seconds
Share: curl http://10.1.10.101:9000/bucket1/ -F bucket=bucket1 -F policy=eyJleHBpcmF0aW9uIjoiMjAyNi0wMS0yNVQyMzoxMjozMS41MDdaIiwiY29uZGl0aW9ucyI6W1siZXEiLCIkYnVja2V0IiwiYnVja2V0MSJdLFsiZXEiLCIka2V5IiwidGVzdC50eHQiXSxbImVxIiwiJHgtYW16LWRhdGUiLCIyMDI2MDExOFQyMzEyMzFaIl0sWyJlcSIsIiR4LWFtei1hbGdvcml0aG0iLCJBV1M0LUhNQUMtU0hBMjU2Il0sWyJlcSIsIiR4LWFtei1jcmVkZW50aWFsIiwiYWRtaW4vMjAyNjAxMTgvdXMtZWFzdC0xL3MzL2F3czRfcmVxdWVzdCJdXX0= -F x-amz-algorithm=AWS4-HMAC-SHA256 -F x-amz-credential=admin/20260118/us-east-1/s3/aws4_request -F x-amz-date=20260118T231231Z -F x-amz-signature=e27668b4822f2bff1292609cd7d27e24134780e787ec376e5d8d7daef1972e62 -F key=test.txt -F file=@<FILE>
```

You can then copy and paste the given curl command and reuse it as much as you want to upload the test.txt object to bucket1 during the allowed period of 7 days.
:warning:
> Remember, attackers can host malwares or overwrite existing files with malicious contents.


a pre-signed URL can be created for GET requests
```shell
mc share download minio-101/bucket1/exercice1/object1.txt --expire 168h
URL: http://10.1.10.101:9000/bucket1/exercice1/object1.txt
Expire: 7 days 0 hours 0 minutes 0 seconds
Share: http://10.1.10.101:9000/bucket1/exercice1/object1.txt?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=admin%2F20260118%2Fus-east-1%2Fs3%2Faws4_request&X-Amz-Date=20260118T231730Z&X-Amz-Expires=604800&X-Amz-SignedHeaders=host&X-Amz-Signature=da6ff503efe4a6ae649adb81105e90fb231a5c61789b6f0b1b80ddb1f73f8eb3
```

The curl command provided can be used in a script, to a user or just share the hashed header in a GET request code but it will be valid only for the specified amount of time.

<br>

Note:
> certain S3 client and servers allow pre-signed POST to allow the upload of any objects on a specified bucket for a certain period of time (max. 7 days)



## 8.2 Risks of Pre-Signed URLs
pre-signed URLs have several inherent risks:
- they don't have any size limit
- the expiration date is long
- there is no content validation
- they are oftenly shared with less caution thans sharing credentials or TLS keys because there are usually no formal processes so they are shared with less precautions.


## 8.3 Blocking or limiting Pre-Signed URLs using BIG-IP iRules

### 8.3.1 Creating an iRule

The following iRule blocks **SigV4 pre-signed URLs** for a **configurable list of S3 buckets**, while allowing:
- Normal authenticated S3 requests (Authorization header)
- Non-pre-signed access to public buckets

<br>
<br>
---

### 8.3.2 How pre-signed URLs are detected

A SigV4 pre-signed URL always contains one or more of the following query parameters:

- `X-Amz-Signature`
- `X-Amz-Credential`
- `X-Amz-Expires`

The iRule checks for these parameters to reliably identify pre-signed requests.

<br>
<br>
---

### 8.3.3 Prerequisites: 

First, we need to create a **string data group** that will contain the list of buckets where we want to deny pre-signed access:

- **Name:** `dg_presign_block_buckets`
- **Type:** String
- **Entries (keys only):**

```text
  finance
  backups
  private-data
  logs
```

and the iRule:
```tcl
when HTTP_REQUEST {
TBD
}
```

## 8.4 Key takeaways
S3 Pre-signed URLs are a great feature that are very convenient for time limited access to a bucket or a specific object. But all great powers should be used carefully! Remember pre-signed URLs are non authenticated endpoints and could be abused, leaked, replayed, or over-privileged.

[Back to Agenda](/README.md)

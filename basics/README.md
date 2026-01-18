# 4. S3 Basics

## 4.0 Introduction

This section of the lab is intended to give you some hands-on experience on S3 bucket and object manipulations. Don't expect to be a Guru in S3 after completed this section, it will give you however the minimum experience to engage a discussion with a Storage Engineer on S3 traffic.

[4.1 Create a Bucket](https://github.com/fchmainy/s3_lab/blob/main/basics/README.md#41-create-a-bucket)

[4.2 Upload an object in the bucket](https://github.com/fchmainy/s3_lab/tree/main/basics#42-upload-an-object-in-the-bucket)

[4.3 Lessons learned](https://github.com/fchmainy/s3_lab/tree/main/basics#lessons-learned)


## 4.1 Create a bucket

### 4.1.1 Introduction
Consider Buckets as Global Namespaces inside a tenant.

### 4.1.2 Using Minio UI
On the UDF Deployment, on the **Access** tab of the server, you will find the following access methods:
- minio-101-console
- minio-102-console
- minio-103-console
- minio-104-console
- minio-105-console

For each of them, or you can mix between UI and CLI if you want this step to be less boring:
login as **admin** with password **MinioPass12345**

on the left side menu click **bucket** > **add**

and create bucket **bucket1**

### 4.1.3 Using CLI: Minio Client or mc
SSH on the client, it already has the mc client installed.

First, we need to create aliases for each nodes, so the mc (minio client) name the nodes and associate each of them with an admin access and secret keys.

```shell
MINIO_ACCESS_KEY="admin"
MINIO_SECRET_KEY="myPass12345"
mc alias set minio-101 http://10.1.10.101:9000   "$MINIO_ACCESS_KEY" "$MINIO_SECRET_KEY"
mc alias set minio-102 http://10.1.10.102:9000   "$MINIO_ACCESS_KEY" "$MINIO_SECRET_KEY"
mc alias set minio-103 http://10.1.10.103:9000   "$MINIO_ACCESS_KEY" "$MINIO_SECRET_KEY"
mc alias set minio-104 http://10.1.10.104:9000   "$MINIO_ACCESS_KEY" "$MINIO_SECRET_KEY"
mc alias set minio-105 http://10.1.10.105:9000   "$MINIO_ACCESS_KEY" "$MINIO_SECRET_KEY"
mc alias set demovip https://mixed.s3.f5demo.com "$MINIO_ACCESS_KEY" "$MINIO_SECRET_KEY" --insecure
```

:notebook:
> the last command creates an alias for the Virtual Server created to serve all nodes. The command has the **--insecure** argument because it is served through HTTPS with a self-signed certificate.


Then, we are creating a bucket named **bucket1** on each of them
```shell
mc mb minio-101/bucket1
mc mb minio-102/bucket1
mc mb minio-103/bucket1
mc mb minio-104/bucket1
mc mb minio-105/bucket1
```

now, you are ready to add objects to your bucket.



## 4.2 Upload an object in the bucket

create a text file locally and upload it:
```shell
echo "This is an object for Bucket1" > /tmp/object1.txt
```
<br>
now you can upload this object into your buckets:
```shell
mc cp /tmp/object1.txt minio-101/bucket1/exercice1/object1.txt
```

**object1.txt** is the file and **exercice1** is its folder. The **object** is identified as the **folder/file**

<br>
Let's debug this request so we can understand what the mc client did here:
We are going to add the **debug** attribute to the request so we can see the HTTP requests issued between the client and the S3 node.

```yaml
mc --debug cp ./object1.txt minio-101/bucket1/exercice1/object1.txt
 0 B / ? ┃░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░▓┃
mc: <DEBUG> GET /bucket1/?location= HTTP/1.1
Host: 10.1.10.101:9000
User-Agent: MinIO (linux; amd64) minio-go/v7.0.90 mc/DEVELOPMENT.GOGET
Accept-Encoding: zstd,gzip
Authorization: AWS4-HMAC-SHA256 Credential=admin/20260113/us-east-1/s3/aws4_request, SignedHeaders=host;x-amz-content-sha256;x-amz-date, Signature=**REDACTED**
X-Amz-Content-Sha256: e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855
X-Amz-Date: 20260113T111334Z

mc: <DEBUG> HTTP/1.1 200 OK
Content-Length: 128
Accept-Ranges: bytes
Content-Type: application/xml
Date: Tue, 13 Jan 2026 11:13:34 GMT
Server: MinIO
Strict-Transport-Security: max-age=31536000; includeSubDomains
Vary: Origin
Vary: Accept-Encoding
X-Amz-Id-2: dd9025bab4ad464b049177c95eb6ebf374d3b3fd1af9251148b658df7ac2e3e8
X-Amz-Request-Id: 188A460A76C66E97
X-Content-Type-Options: nosniff
X-Ratelimit-Limit: 6974
X-Ratelimit-Remaining: 6974
X-Xss-Protection: 1; mode=block

mc: <DEBUG> Response Time:  3.211548ms

mc: <DEBUG> GET /bucket1/?object-lock= HTTP/1.1
Host: 10.1.10.101:9000
User-Agent: MinIO (linux; amd64) minio-go/v7.0.90 mc/DEVELOPMENT.GOGET
Accept-Encoding: zstd,gzip
Authorization: AWS4-HMAC-SHA256 Credential=admin/20260113/us-east-1/s3/aws4_request, SignedHeaders=host;x-amz-content-sha256;x-amz-date, Signature=**REDACTED**
X-Amz-Content-Sha256: e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855
X-Amz-Date: 20260113T111334Z

mc: <DEBUG> HTTP/1.1 404 Not Found
Content-Length: 360
Accept-Ranges: bytes
Content-Type: application/xml
Date: Tue, 13 Jan 2026 11:13:34 GMT
Server: MinIO
Strict-Transport-Security: max-age=31536000; includeSubDomains
Vary: Origin
Vary: Accept-Encoding
X-Amz-Id-2: dd9025bab4ad464b049177c95eb6ebf374d3b3fd1af9251148b658df7ac2e3e8
X-Amz-Request-Id: 188A460A76EC5745
X-Content-Type-Options: nosniff
X-Ratelimit-Limit: 6974
X-Ratelimit-Remaining: 6974
X-Xss-Protection: 1; mode=block

<?xml version="1.0" encoding="UTF-8"?>
<Error><Code>ObjectLockConfigurationNotFoundError</Code><Message>Object Lock configuration does not exist for this bucket</Message><BucketName>bucket1</BucketName><Resource>/bucket1/</Resource><RequestId>188A460A76EC5745</RequestId><HostId>dd9025bab4ad464b049177c95eb6ebf374d3b3fd1af9251148b658df7ac2e3e8</HostId></Error>mc: <DEBUG> Response Time:  1.4626ms

mc: <DEBUG> HEAD /bucket1/exercice1/object1.txt HTTP/1.1
Host: 10.1.10.101:9000
User-Agent: MinIO (linux; amd64) minio-go/v7.0.90 mc/DEVELOPMENT.GOGET
Authorization: AWS4-HMAC-SHA256 Credential=admin/20260113/us-east-1/s3/aws4_request, SignedHeaders=host;x-amz-checksum-mode;x-amz-content-sha256;x-amz-date, Signature=**REDACTED**
X-Amz-Checksum-Mode: ENABLED
X-Amz-Content-Sha256: e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855
X-Amz-Date: 20260113T111334Z

mc: <DEBUG> HTTP/1.1 404 Not Found
Accept-Ranges: bytes
Date: Tue, 13 Jan 2026 11:13:34 GMT
Server: MinIO
Strict-Transport-Security: max-age=31536000; includeSubDomains
Vary: Origin
Vary: Accept-Encoding
X-Amz-Id-2: dd9025bab4ad464b049177c95eb6ebf374d3b3fd1af9251148b658df7ac2e3e8
X-Amz-Request-Id: 188A460A7713E0C0
X-Content-Type-Options: nosniff
X-Minio-Error-Code: NoSuchKey
X-Minio-Error-Desc: "The specified key does not exist."
X-Ratelimit-Limit: 6974
X-Ratelimit-Remaining: 6974
X-Xss-Protection: 1; mode=block
Content-Length: 0

mc: <DEBUG> Response Time:  1.874234ms

mc: <DEBUG> GET /bucket1/?delimiter=%2F&encoding-type=url&fetch-owner=true&list-type=2&max-keys=1&prefix=exercice1%2Fobject1.txt%2F HTTP/1.1
Host: 10.1.10.101:9000
User-Agent: MinIO (linux; amd64) minio-go/v7.0.90 mc/DEVELOPMENT.GOGET
Accept-Encoding: zstd,gzip
Authorization: AWS4-HMAC-SHA256 Credential=admin/20260113/us-east-1/s3/aws4_request, SignedHeaders=host;x-amz-content-sha256;x-amz-date, Signature=**REDACTED**
X-Amz-Content-Sha256: e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855
X-Amz-Date: 20260113T111334Z

mc: <DEBUG> HTTP/1.1 200 OK
Content-Length: 313
Accept-Ranges: bytes
Content-Type: application/xml
Date: Tue, 13 Jan 2026 11:13:34 GMT
Server: MinIO
Strict-Transport-Security: max-age=31536000; includeSubDomains
Vary: Origin
Vary: Accept-Encoding
X-Amz-Id-2: dd9025bab4ad464b049177c95eb6ebf374d3b3fd1af9251148b658df7ac2e3e8
X-Amz-Request-Id: 188A460A773B852B
X-Content-Type-Options: nosniff
X-Ratelimit-Limit: 6974
X-Ratelimit-Remaining: 6974
X-Xss-Protection: 1; mode=block

mc: <DEBUG> Response Time:  2.428734ms

mc: <DEBUG> PUT /bucket1/exercice1/object1.txt HTTP/1.1
Host: 10.1.10.101:9000
User-Agent: MinIO (linux; amd64) minio-go/v7.0.90 mc/DEVELOPMENT.GOGET
Content-Length: 203
Accept-Encoding: zstd,gzip
Authorization: AWS4-HMAC-SHA256 Credential=admin/20260113/us-east-1/s3/aws4_request,SignedHeaders=host;x-amz-content-sha256;x-amz-date;x-amz-decoded-content-length,Signature=**REDACTED**
Content-Type: text/plain
X-Amz-Content-Sha256: STREAMING-AWS4-HMAC-SHA256-PAYLOAD
X-Amz-Date: 20260113T111334Z
X-Amz-Decoded-Content-Length: 30

mc: <DEBUG> HTTP/1.1 200 OK
Content-Length: 0
Accept-Ranges: bytes
Date: Tue, 13 Jan 2026 11:13:34 GMT
Etag: "723f593bf8031534ee9d9d6983e37f3e"
Server: MinIO
Strict-Transport-Security: max-age=31536000; includeSubDomains
Vary: Origin
Vary: Accept-Encoding
X-Amz-Id-2: dd9025bab4ad464b049177c95eb6ebf374d3b3fd1af9251148b658df7ac2e3e8
X-Amz-Request-Id: 188A460A776D5E7F
X-Content-Type-Options: nosniff
X-Ratelimit-Limit: 6974
X-Ratelimit-Remaining: 6974
X-Xss-Protection: 1; mode=block

mc: <DEBUG> Response Time:  4.950594ms

/tmp/object1.txt:                     30 B / 30 B
```

<br>
Let's see in details what happened:

- GET /bucket1/?location= (200 OK)
  mc checks the bucket’s region (S3 LocationConstraint).
  
- GET /bucket1/?object-lock= (404 Not Found)
  mc checks if Object Lock is enabled; bucket has no object-lock config.
  
- HEAD /bucket1/exercice1/object1.txt (404 Not Found - NoSuchKey)
  mc verifies whether the target object already exists.
  
- GET /bucket1/?list-type=2&prefix=exercice1/object1.txt/ (200 OK)
  mc ensures no conflicting “directory-like” prefix exists.
  
- PUT /bucket1/exercice1/object1.txt (streaming payload) (200 OK)
  The object is uploaded successfully; MinIO returns an ETag.

Result: upload completes, 30 B / 30 B transferred successfully.

<br><br>

The 3 first requests are for metadata control (region, object lock, head and get). For an upload, the most important request is the last one **PUT**.
<br><br>
But the most interesting part to look into is the Authorization header. Remember, here we use credentials (ACCESS_KEY and SECRET_KEY):
- **Authorization: AWS4-HMAC-SHA256…** indicates Signature Version 4 (SigV4).
- **Credential=admin/20260113/us-east-1/s3/aws4_request** Access key is **admin**, date of the request is **20260113**, the region of the node is **us-east-1**.
- **SignedHeaders=host;x-amz-content-sha256;x-amz-date[;…]** List of the HTTP headers that are signed.
- **Signature=<HMAC-SHA256>** HMAC over the canonical request, using the derived signing key.
  The Canonical request includes: HTTP method, URI, query string, signed headers, and payload hash.
- For metadata calls (GET/HEAD): **X-Amz-Content-Sha256 = e3b0c4…** which is the SHA256 of an empty body.
- For the PUT upload: **X-Amz-Content-Sha256 = STREAMING-AWS4-HMAC-SHA256-PAYLOAD**
- **X-Amz-Date: 20260113T111334Z** Request timestamp used in key derivation and replay protection.
- **X-Amz-Decoded-Content-Length: 30** Original payload size before streaming/chunk encoding.

When the objects arrive on the S3 node, the S3 service recomputes the signature server-side and compares it with the received hash; if the hashes match the access is allowed. If not, the server issues a **403 SignatureDoesNotMatch** or **AccessDenied** response.

You have more details on the Canonical hash at: [https://docs.aws.amazon.com/AmazonS3/latest/API/sig-v4-header-based-auth.html](https://docs.aws.amazon.com/AmazonS3/latest/API/sig-v4-header-based-auth.html)

<br><br>

:warning:Notes
>  - Signatures embeds permissions and are time limited. Each request is authentified using the user SECRET_KEY.
>  -  SECRET_KEYS are never supposed to be sent over the wire. Only the user ACCESS_KEY (~username).


## 4.3 Download an object in the bucket

1. You can list the content of a bucket and its folders:
   
```shell
mc ls minio-101/bucket1/
[2026-01-13 14:48:36 UTC]     0B exercice1/

mc ls minio-101/bucket1/exercice1
[2026-01-13 11:13:34 UTC]    30B STANDARD object1.txt
```

2. you can get all metadata of an object:

```shell
mc stat minio-101/bucket1/exercice1/object1.txt
Name      : object1.txt
Date      : 2026-01-13 11:13:34 UTC
Size      : 30 B
ETag      : 723f593bf8031534ee9d9d6983e37f3e
Type      : file
Metadata  :
  Content-Type: text/plain
```

3. Display the first lines of an object (default is 10 lines, you can specifify how many lines you want to print):

```shell
mc head --lines 5 minio-101/bucket1/exercice1/object1.txt
This is an object for Bucket1
```

4. You can download the file to a specific local emplacement:

```shell
mc get minio-101/bucket1/exercice1/object1.txt /tmp/object1_downloaded.txt
...//10.1.10.101:9000/bucket1/exercice1/object1.txt: 30 B / 30 B ┃▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓┃ 3.00 KiB/s 0subuntu@client:~$ cat /tmp/object1_downloaded.txt
This is an object for Bucket1
```
Now, you can run these commands with the **debug** argument and you will see the headers and the canonical hashed headers

```shell
mc: <DEBUG> GET /bucket1/exercice1/object1.txt HTTP/1.1
Host: 10.1.10.101:9000
User-Agent: MinIO (linux; amd64) minio-go/v7.0.90 mc/DEVELOPMENT.GOGET
Accept-Encoding: identity
Authorization: AWS4-HMAC-SHA256 Credential=admin/20260113/us-east-1/s3/aws4_request, SignedHeaders=host;x-amz-content-sha256;x-amz-date, Signature=**REDACTED**
X-Amz-Content-Sha256: e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855
X-Amz-Date: 20260113T145807Z

mc: <DEBUG> HTTP/1.1 200 OK
Content-Length: 30
Accept-Ranges: bytes
Content-Type: text/plain
Date: Tue, 13 Jan 2026 14:58:07 GMT
Etag: "723f593bf8031534ee9d9d6983e37f3e"
Last-Modified: Tue, 13 Jan 2026 11:13:34 GMT
Server: MinIO
Strict-Transport-Security: max-age=31536000; includeSubDomains
Vary: Origin
Vary: Accept-Encoding
X-Amz-Id-2: dd9025bab4ad464b049177c95eb6ebf374d3b3fd1af9251148b658df7ac2e3e8
X-Amz-Request-Id: 188A524B55213341
X-Content-Type-Options: nosniff
X-Ratelimit-Limit: 6974
X-Ratelimit-Remaining: 6974
X-Xss-Protection: 1; mode=block

mc: <DEBUG> Response Time:  2.173715ms

This is an object for Bucket1
```

<br><br>
## 4.3 What if I don't want to use an S3 client?

If you try accessing your object through HTTP using curl, it should fail:
```shell
curl -vvv http://10.1.10.101:9000/bucket1/exercice1/object1.txt
*   Trying 10.1.10.101:9000...
* Connected to 10.1.10.101 (10.1.10.101) port 9000 (#0)
> GET /bucket1/exercice1/object1.txt HTTP/1.1
> Host: 10.1.10.101:9000
> User-Agent: curl/7.81.0
> Accept: */*
>
* Mark bundle as not supporting multiuse
< HTTP/1.1 403 Forbidden
< Accept-Ranges: bytes
< Content-Length: 347
< Content-Type: application/xml
< Server: MinIO
< Strict-Transport-Security: max-age=31536000; includeSubDomains
< Vary: Origin
< Vary: Accept-Encoding
< X-Amz-Id-2: dd9025bab4ad464b049177c95eb6ebf374d3b3fd1af9251148b658df7ac2e3e8
< X-Amz-Request-Id: 188A5288B4C92B23
< X-Content-Type-Options: nosniff
< X-Ratelimit-Limit: 6974
< X-Ratelimit-Remaining: 6974
< X-Xss-Protection: 1; mode=block
< Date: Tue, 13 Jan 2026 15:02:30 GMT
<
<?xml version="1.0" encoding="UTF-8"?>
* Connection #0 to host 10.1.10.101 left intact
<Error><Code>AccessDenied</Code><Message>Access Denied.</Message><Key>exercice1/object1.txt</Key><BucketName>bucket1</BucketName><Resource>/bucket1/exercice1/object1.txt</Resource><RequestId>188A5288B4C92B23</RequestId><HostId>dd9025bab4ad464b049177c95eb6ebf374d3b3fd1af9251148b658df7ac2e3e8</HostId></Error
```
You get a **403 - Forbidden** response with an **Access Denied** message because S3 Services are authenticated by default (least privileges).

:notebook:
> The same principles exists when building an app code that requests to access objects in S3 buckets. You can either:
> - use a SDK (boto3, minio-py...)
> - build your own functions to compute the canonical hash.

<br><br>
or...
You can make the bucket public by setting an anonymous access to it:

```shell
mc anonymous set public minio-101/bucket1
```

By setting the bucket anonymous, you have removed the authentication entirely.

```shell
curl http://10.1.10.101:9000/bucket1/
<?xml version="1.0" encoding="UTF-8"?>
<ListBucketResult xmlns="http://s3.amazonaws.com/doc/2006-03-01/"><Name>bucket1</Name><Prefix></Prefix><Marker></Marker><MaxKeys>1000</MaxKeys><IsTruncated>false</IsTruncated><Contents><Key>exercice1/object1.txt</Key><LastModified>2026-01-13T11:13:34.464Z</LastModified><ETag>&#34;723f593bf8031534ee9d9d6983e37f3e&#34;</ETag><Size>30</Size><Owner><ID>02d6176db174dc93cb1b899f7c6078f08654445fe8cf1b6ce98d8855f66bdbf4</ID><DisplayName>minio</DisplayName></Owner><StorageClass>STANDARD</StorageClass></Contents></ListBucketResult>

curl -vvv http://10.1.10.101:9000/bucket1/exercice1/object1.txt
*   Trying 10.1.10.101:9000...
* Connected to 10.1.10.101 (10.1.10.101) port 9000 (#0)
> GET /bucket1/exercice1/object1.txt HTTP/1.1
> Host: 10.1.10.101:9000
> User-Agent: curl/7.81.0
> Accept: */*
>
* Mark bundle as not supporting multiuse
< HTTP/1.1 200 OK
< Accept-Ranges: bytes
< Content-Length: 30
< Content-Type: text/plain
< ETag: "723f593bf8031534ee9d9d6983e37f3e"
< Last-Modified: Tue, 13 Jan 2026 11:13:34 GMT
< Server: MinIO
< Strict-Transport-Security: max-age=31536000; includeSubDomains
< Vary: Origin
< Vary: Accept-Encoding
< X-Amz-Id-2: dd9025bab4ad464b049177c95eb6ebf374d3b3fd1af9251148b658df7ac2e3e8
< X-Amz-Request-Id: 188A52C651B6FB0E
< X-Content-Type-Options: nosniff
< X-Ratelimit-Limit: 6974
< X-Ratelimit-Remaining: 6974
< X-Xss-Protection: 1; mode=block
< Date: Tue, 13 Jan 2026 15:06:55 GMT
<
This is an object for Bucket1
* Connection #0 to host 10.1.10.101 left intact
```

but you can also add objects:
```shell
echo "This is a second object for Bucket1" > /tmp/object2.txt
curl -X PUT -d@/tmp/object2.txt http://10.1.10.101:9000/bucket1/exercice1/object2.txt -vvv
*   Trying 10.1.10.101:9000...
* Connected to 10.1.10.101 (10.1.10.101) port 9000 (#0)
> PUT /bucket1/exercice1/object2.txt HTTP/1.1
> Host: 10.1.10.101:9000
> User-Agent: curl/7.81.0
> Accept: */*
> Content-Length: 35
> Content-Type: application/x-www-form-urlencoded
>
* Mark bundle as not supporting multiuse
< HTTP/1.1 200 OK
< Accept-Ranges: bytes
< Content-Length: 0
< ETag: "75afa505f61b67b388d19708b370482d"
< Server: MinIO
< Strict-Transport-Security: max-age=31536000; includeSubDomains
< Vary: Origin
< Vary: Accept-Encoding
< X-Amz-Id-2: dd9025bab4ad464b049177c95eb6ebf374d3b3fd1af9251148b658df7ac2e3e8
< X-Amz-Request-Id: 188A52EF50CBB350
< X-Content-Type-Options: nosniff
< X-Ratelimit-Limit: 6974
< X-Ratelimit-Remaining: 6974
< X-Xss-Protection: 1; mode=block
< Date: Tue, 13 Jan 2026 15:09:51 GMT
<
* Connection #0 to host 10.1.10.101 left intact

curl http://10.1.10.101:9000/bucket1/
<?xml version="1.0" encoding="UTF-8"?>
<ListBucketResult xmlns="http://s3.amazonaws.com/doc/2006-03-01/"><Name>bucket1</Name><Prefix></Prefix><Marker></Marker><MaxKeys>1000</MaxKeys><IsTruncated>false</IsTruncated><Contents><Key>exercice1/object1.txt</Key><LastModified>2026-01-13T11:13:34.464Z</LastModified><ETag>&#34;723f593bf8031534ee9d9d6983e37f3e&#34;</ETag><Size>30</Size><Owner><ID>02d6176db174dc93cb1b899f7c6078f08654445fe8cf1b6ce98d8855f66bdbf4</ID><DisplayName>minio</DisplayName></Owner><StorageClass>STANDARD</StorageClass></Contents><Contents><Key>exercice1/object2.txt</Key><LastModified>2026-01-13T15:09:51.503Z</LastModified><ETag>&#34;75afa505f61b67b388d19708b370482d&#34;</ETag><Size>35</Size><Owner><ID>02d6176db174dc93cb1b899f7c6078f08654445fe8cf1b6ce98d8855f66bdbf4</ID><DisplayName>minio</DisplayName></Owner><StorageClass>STANDARD</StorageClass></Contents></ListBucketResult>
```

or even worst:
```shell
curl -X DELETE http://10.1.10.101:9000/bucket1/exercice1/object2.txt

mc ls minio-101/bucket1/exercice1
[2026-01-13 11:13:34 UTC]    30B STANDARD object1.txt
```


<br><br>
## 4.3 Lessons learned
- Authorization happens on the server side.
- **making a bucket public (anonymous) removes authentication entirely on the bucket** and does not require credentials or signature.

Although it makes life easier, <font color="red">**you should never make buckets public**</font> because it turns your s3 storage into a possible attacker infrastructure: not only for data leakage risks, but also could be used for C&C and as an exfiltration platform:
- allow non authorized users accessing confidential objects (containing Intellectual Property, PII...)
- allow malicious users delete objects
- attackers can host malwares
- allow malicious users create ransomware by encrypting objects.

all of these with legal & reputational impacts.

<br><br>

As a reference, please find the MITRE ATT&CK techniques related to public buckets.

| MITRE ATT&CK ID | Technique Name                                | Tactic              | How It Relates to Public S3 Buckets | Real-World Incident Example |
|----------------|----------------------------------------------|---------------------|------------------------------------|-----------------------------|
| T1530          | Data from Cloud Storage                       | Collection          | Attackers list and download data from publicly accessible S3 buckets without authentication. | 2017–2019: Multiple breaches (Verizon, Accenture, Dow Jones) where sensitive data was harvested from publicly exposed S3 buckets via anonymous access. |
| T1567.002      | Exfiltration Over Web Services: Cloud Storage | Exfiltration        | Stolen data is uploaded to attacker-controlled S3 buckets to blend in with legitimate cloud traffic. | FIN7 and Magecart groups used attacker-owned S3 buckets to exfiltrate stolen payment card data over HTTPS. |
| T1102.003      | Web Service: Cloud Storage                    | Command and Control | Public S3 buckets are used as C2 infrastructure for malware command retrieval. | APT29 (Cozy Bear) used AWS S3 buckets as dead-drop resolvers for command-and-control in multiple espionage campaigns. |
| T1105          | Ingress Tool Transfer                         | Command and Control | Malware, scripts, or payloads are anonymously downloaded from public S3 buckets. | Emotet and TrickBot campaigns hosted payloads in public S3 buckets to evade takedowns and filtering. |
| T1071.001      | Application Layer Protocol: Web Protocols     | Command and Control | Anonymous S3 access uses HTTPS, making malicious traffic appear legitimate. | Cobalt Strike beacons observed downloading profiles and payloads over HTTPS from public cloud storage endpoints including S3. |
| T1036          | Masquerading                                  | Defense Evasion     | Traffic to public S3 buckets blends into normal enterprise cloud usage. | Red Team and real-world intrusions documented by Mandiant show malware using S3 domains to evade proxy allowlists. |
| T1074.001      | Data Staged: Local Data Staging               | Collection          | Public buckets serve as intermediate storage before final exfiltration. | Lazarus Group staged collected data in temporary S3 buckets before moving it to long-term infrastructure. |
| T1496          | Resource Hijacking                            | Impact              | Public buckets distribute cryptominers or large-scale malicious tooling. | Cryptojacking campaigns (e.g., Graboid) used public S3 buckets to host miners and orchestration scripts. |

<br><br>

[Back to Agenda](/README.md)

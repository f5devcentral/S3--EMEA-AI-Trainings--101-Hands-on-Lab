# 5. BIG-IP Configuration

## 5.1 TCP Profile

TMOS v21.0 introduces a new TCP Profile called **s3-tcp** customized to optimize S3 performances and provide pre-configured settings including:
- enhanced buffer management, 
- optimized congestion control, 
- high-throughput data transfer capabilities. 

This profiles integrates as one-click default options within the LTM configuration, offering customers an intuitive, streamlined experience with reduced complexity while maximizing performance and scalability for S3 operations.


<p align="center">
	<img width="500" src="https://github.com/fchmainy/s3_lab/blob/main/docs/s3_lab_s3-tcp_profile.jpg?raw=true" alt="UDF S3 Topo">
</p>
<br>
<br>
<br>


## 5.2 TLS Profile

TMOS v21.0 also introduces a new Client SSL Profile **s3-default-clientssl** that brings a pre-configured settings specifically tuned for S3 efficient SSL session handling. 

This profiles enables a seamless TLS offload, improve encrypted communication performance, and integrates as one-click default options within the LTM configuration.

<br>
<br>

## 5.3 CARP Persistence

CARP (Cache Array Routing Protocol) is a deterministic hashing algorithm used to map a URI path to the same backend node.

As you saw previously, S3 is not a stateless HTTP workload and the bucket and object names (including object versions) are located in the path and Without persistence, S3 breaks.

 - **Same cluster, Erasure Set:**
Usually, in a single cluster operation, with Erasure Coding set as your default replication strategy, when you PUT an object on a node and you request the same object on the same bucket on a different node (because you do not have a proper persistence configuration), it does not matter, you don't need any persistence:: the object is rebuilt from all the shards before delivering it.
<br>
 - **Same cluster, Replication (n-way copies):**
When you do replication, ach object is stored as full copies across multiple nodes/disks (x2 or xN copies). Then, the fastest way to get the object is konwing where you uploaded it. If you are load balanced on a different node within the cluster, the object is first copied locally on the node you landed on before delivering to you which adds more or less latency based on the object size.
<br>
 - **Multipart PUT**
Multipart uploads and downloads require persistence.
We will see later in the lab the details on multipart and why it makes sense to have CARP persistence on it
[S3 Multipart](https://github.com/fchmainy/s3_lab/tree/main/multipart%20#7-s3-multipart)

<br>
<br>
** Configuration of CARP**
CARP is not new, the configuration for S3 is an adaptation of what we already had for decades (https://my.f5.com/manage/s/article/K11362)
<br>
<ins>Step1: Create the CARP iRule</ins>

Local Traffic >> iRule

```python
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
<br><br>
<ins>Step2: Create the CARP Persistence Profile</ins>
Create a Persistence Profile in Local Traffic >> Profiles >> Persistence
Persistence Type is **Hash**

in the **Configuration** section, override :
- the **Hash Algorithm** and choose **CARP** in the dropdown list
- the **iRule** and choose the iRule you previsouly created.

Leave every other setting default.
<br><br>
<ins>Step3: Attach the Persistence Profile to your Virtual Server</ins>
Local Traffic >> Virtual Servers : Virtual Server List >> read.s3.f5demo.com
On the **Resources** tab, choose your CARP Persistence Profile in the **Default Persistence Profile** dropdown list.

You can add a **source_addr** as a **Fallback Persistence Profile**.
<br>
<br>

## 5.4 OneConnect

Now that we have our CARP persistence set, we can add a OneConnect profile.
OneConnect, like any HTTP connection reuse mechanism, is generally not recommended when accessing AWS S3, because S3 is delivered as a fully managed service and you have no control over backend node affinity, connection handling, or request-to-node mapping. Reusing client-side connections can lead to unpredictable behavior, uneven load distribution, or reduced performance, since successive requests over the same connection may be handled by different internal components of the S3 service.

In contrast, when using self-managed S3-compatible clusters, such as MinIO, Dell ECS, NetApp, or any other self-hosted S3 services, enabling OneConnect is highly recommended. 

The difference with AWS S3, when you manage the S3 storage infrastructure, you control:
 - the backend nodes topology,
 - the load-balancing algorithms,
 - the persistence like CARP
 - the Health monitors.

Then OneConnect adds the following benefits:
 - Reduce TCP/TLS connection overhead
 - Improve backend node utilization
 - Increase throughput and lower latency

<p align="center">
	<img width="500" src="https://github.com/fchmainy/s3_lab/blob/main/docs/oneConnect.jpg?raw=true" alt="One Connect Menu">
</p>
<br>

First, we want to keep a fair number of maximum reuse, especially if you are dealing with large objects (>1GB) uploads so we do not introduce uneven nodes pressure. Let's take 20 and if the objects are smaller we could increase this number progressively.
Second, let's prevents long-lived idle connections pinning backend nodes

Ensures connection reuse is per client: **255.255.255.255**

<br>
<p align="center">
	<img width="500" src="https://github.com/fchmainy/s3_lab/blob/main/docs/oneconnect_config.jpg?raw=true" alt="One Connect Configuration">
</p>
<br>
<br>

## 5.4 Monitors
In order to keep a health-check of the S3 nodes there are 3 ways we can have

**a) Use the publicly accessible health or ready endpoints**

```shell
create ltm monitor https minio_s3_ready_monitor {
    defaults-from http
    interval 10
    timeout 31
    send "GET /minio/health/ready\r\n"
    recv "200 OK"
    recv-disable "403|404|500"
}
```

**b) Use a dummy anonymous bucket/object**
This method is more realistic because each monitor will perform real S3 requests.
of course we don't want to store credentials on the BIG-IP for two obvious reasons:

- store credentials in clear text
- if you want to change your passwords because it your security policy mandates to do so, then you will have all your monitors failing.
<br>
So, forget what I have said about public buckets and let's create one:

```shell
mc mb minio-101/monitor
mc mb minio-102/monitor
mc mb minio-103/monitor
mc mb minio-104/monitor
mc mb minio-105/monitor

echo "OK" > monitor.txt

mc cp ~/monitor.txt minio-101/monitor/monitor
mc cp ~/monitor.txt minio-102/monitor/monitor
mc cp ~/monitor.txt minio-103/monitor/monitor
mc cp ~/monitor.txt minio-104/monitor/monitor
mc cp ~/monitor.txt minio-105/monitor/monitor

mc anonymous set public minio-101/monitor
mc anonymous set public minio-102/monitor
mc anonymous set public minio-103/monitor
mc anonymous set public minio-104/monitor
mc anonymous set public minio-105/monitor
```


<br>
then configure the monitor on the BIG-IP:

```shell
create ltm monitor http minio_s3_object_monitor {
    defaults-from http
    interval 10
    timeout 31
    send "GET /monitor/monitor.txt HTTP/1.1\r\n"
    recv "OK"
}
```
<br><br>
**c) EAV Monitors on prometheus metrics**
The last method we can use for monitoring is using the nodes prometheus endpoints. These are not enabled by default and are outside of S3 specifications so you would need to adapt it depending on the S3 vendor you use.
<br>
We could use an External EAV Monitor that will scrape the prometheus endpoint, get the node CPU level or any other metric or combination of metrics and compute a decision on the health of the node.
<br>
*We will not cover this type of monitor in lab; just keep in mind it is something possible and could answer some advanced monitor use cases.*



### 5.5 AST / F5 Insights
<br>
<font color="green">**Disclaimer:** when I finished the lab AST was still a real thing. I promise I will update the lab once we get F5 Insights (~March 2026).</font>
<br>
As you understand, S3 is a distributed architecture which needs to be load balanced.
As we have BIG-IP as an intermediate proxy that brings a lot of value both in term of delivery and security, we need to show this values by collecting metrics and events.

we will use AST to collect BIG-IP metrics and enrich these metrics using Prometheus to scrape the nodes metrics from the MinIO nodes and build dashboards.

<p align="center">
	<img width="500" src="https://github.com/fchmainy/s3_lab/blob/main/docs/Telemetry_s3_udf.jpg?raw=true" alt="UDF S3 Telemetry">
</p>
<br>
<br>
<br>


We have also provided the following iRule to stream events from the BIG-IP to an OpenTelemtry Collector that will export them into Loki.

on Grafana, we have added a set a second set of dashboards.

<p align="center">
	<img width="500" src="https://github.com/fchmainy/s3_lab/blob/main/docs/s3_bucket_stats.jpg?raw=true" alt="Grafana Bucket Stats">
	<img width="500" src="https://github.com/fchmainy/s3_lab/blob/main/docs/Telemetry_s3_udf.jpg?raw=true" alt="Grafana S3 metrics">
</p>
<br>
<br>

<br><br>
[Back to Agenda](/README.md)

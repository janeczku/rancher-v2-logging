# Rancher 2.5 Logging Guidance

Rancher 2.5 ships with a new logging integration based on the [Banzai Logging Operator](https://github.com/banzaicloud/logging-operator). This repository contains a few
configuration examples implementing recommended settings for production, migration notes as well as best practices and troubleshooting with a focus on Elasticsearch log storage.

**Quick Jump To:**

[Examples](examples/)    

[Migration Notes](#migration-notes)    

[Mastering common logging challenges with Logging v2](#mastering-common-logging-challenges-with-logging-v2)


## Configuration examples

The `examples` folder covers:

- Examples for configuring separate logging scopes for cluster administrators and users
- Best-practice configuration examples for Elasticsearch and S3 integration
- Example of setting up advanced log processing (parse ingress controller access logs to structured data)

## Migration notes

Rancher's new logging integration (aka "Logging v2") provides the user with a lot of control over the entire flow of collecting, processing and log shipping.
The less opinionated configuration framework also means that some aspects of logging may behave differently out of the box, especially when using the Cluster Explorer UI to setup the logging outputs.

Luckily, it's very easy to bring things in line with pre-2.5 logging behaviour by setting the right parameters in the Logging output specs.
Here are a number of gotchas you might run into while migrating to v2 logging (using Elasticsearch as output) and how to address them:

### Logs may not show up in Kibana after migrating to v2 Logging

The Kibana index pattern you might have created for Rancher's v1 Loggging may not capture the v2 Logging indexes since the latter doesn't generate time-formatted indexes by default.
 
Logging v1 generated index names using a pattern of `${prefix}-${dateFormat}`. To retain this naming convention after migration configure the Elasticsearch output as follows:

```
apiVersion: logging.banzaicloud.io/v1beta1
kind: ClusterOutput
metadata:
  name: elasticsearch
  namespace: cattle-logging-system
spec:
  elasticsearch:
    host: elasticsearch-cluster
    (...)
    logstash_format: true
    logstash_prefix: cluster-foo
    logstash_dateformat: '%Y-%m-%d'
    (...)
```

### Fluentd may fail to send logs to Elasticsearch due to permission errors even though the same credentials are used

In Logging v1 the Elasticsearch node discovery was disabled by default (`reload_connection false`) which meant that the user account configured for Elasticsearch output didn't require permissions to query the
Elasticsearch node status API. With Logging v2 that option is not disabled by default and so you may see permission errors
appearing in the logs when the user hasn't been granted the [`monitor` permissions](https://www.elastic.co/guide/en/elasticsearch/reference/current/security-privileges.html) on the Elastic side.

How to fix:    

If the Elasticsearch cluster is behind a reverse proxy, node discovery via the API should be disabled alltogether (see below). Otherwise the required permissions must be granted to the Elasticsearch user as documented [here](https://github.com/uken/fluent-plugin-elasticsearch#elasticsearch-permissions).

### Fluentd may suddenly not be able to send logs to Elasticsearch behind a load balancer due to connection issues

In contrast to Logging v1, the Elasticsearch output in Logging v2 uses by default the endpoint addresses of the individual Elasticsearch cluster nodes to send bulk log data.
This may cause issues when the Elasticsearch cluster is behind a load balancer and Fluentd has no direct connectivity to the individual hosts.

While this can be mitigated by setting `reload_connections` option to `false` (which Logging v1 did by default) the [upstream plugin documentation](https://github.com/uken/fluent-plugin-elasticsearch#sniffer-class-name) actually recommends to override the default sniffer class in this reverse proxy scenario.

```
apiVersion: logging.banzaicloud.io/v1beta1
kind: ClusterOutput
metadata:
  name: elasticsearch
  namespace: cattle-logging-system
spec:
  elasticsearch:
    host: elasticsearch-cluster
    (...)
    sniffer_class_name: "Fluent::Plugin::ElasticsearchSimpleSnifferAdd"
    (...)
```

### ClusterFlows created via the Cluster Explorer seem to have no effect

`ClusterOutput` and `ClusterFlow` resources always need to be created in the `cattle-logging-system` namespace, as otherwise they will be ignored.

While the correct namespace will be selected automatically when configuring a ClusterOutput in the Cluster Explorer UI, that's currently not the case for `ClusterFlow` so extra care needs to be taken when creating those.

## Mastering common logging challenges with Logging v2

Logging v2 makes it very easy to address common challenges of shipping logs from Kubernetes and implement best practices. Specificallyit allwos us to access lower level Fluentd configuration that was 
not exposed in Rancher's previous Logging implementation.

### Fluentd may consume all node disk space

A common problem with Fluentd is that the default file buffer limit is quite large (64GB) and in the event of a prolonged downtime of the upstream logstore or poor flushing performance, Fluentd may fill all available disk space on the node causing disruption for other pods or even critical OS services.

It's therefore a good practice to configure a limit for the file buffer  (`total_limit_size`) that reflects the available space on the storage medium used for the buffer.

```
apiVersion: logging.banzaicloud.io/v1beta1
kind: ClusterOutput
metadata:
  name: elasticsearch
  namespace: cattle-logging-system
spec:
  elasticsearch:
    host: elasticsearch-cluster
    (...)
    buffer:
      type: file
      # Must be lower than the available space of the storage medium,
      # e.g. a PV, hostPath or pod ephemeral storage.
      total_limit_size: 10GB
    (...)
```

### Updating Fluentd may result in loss of log data

When using a buffer of type `file` in your output (as recommended for production) Fluentd won't by default flush the buffer when receiving the SIGTERM signal.

This creates a problem when running Fluentd as a pod and storing the buffer on ephemeral storage (e.g. `emptyDir`) as you will loose all the contents of the buffer
when the pod instance is terminated.

You can explicitely set `flush_at_shutdown true` in the buffer options, but that does not guarantee that everything is flushed as this process may take longer than the configured pod termination grace period.
Instead, the recommended practice is to store buffer contents on a persistent volume or hostPath volume (see below) so that it's retention is not bound to the lifecycle of the pod.

### Fluentd may drain I/O resources and impact performance of other workloads

Buffering may consume considerable I/O resources depending on log volumes. It's best practice to store fluentd buffers on a dedicated storage medium (via a Persistent Volume or HostPath) and thus eliminate
the noisy neighbour problem for IO resources on your worker nodes.
   
The Fluentd instance created by Rancher Logging v2 uses an `emptyDir` volume for buffer storage by default, meaning that it will share the performance of the Kubernetes ephemeral storage medium with all other pods.
As of now, the `rancher-logging` app doesn't yet allow us to configure a custom buffer location for the default Logging instance. This leaves two options:

- Create a new Logging instance (which will operate on Logging resources its own control namespace)
- Patch the default Logging resource named "rancher-logging" (changes may not persist across updates of the logging app)

To store buffers on a dedicated Persistent Volume you will need to add a `bufferStorageVolume` stanze to the Logging resource like so:

```
apiVersion: logging.banzaicloud.io/v1beta1
kind: Logging
metadata:
  name:logging-with-dedicated-buffer-storage
spec:
  controlNamespace: dedicated-namespace
  fluentd: 
    disablePvc: false
    bufferStorageVolume:
      pvc:
        spec:
          accessModes:
            - ReadWriteOnce
          resources:
            requests:
              storage: 10Gi
          storageClassName: longhorn
  fluentbit: {}
```

See [upstream docs](https://banzaicloud.com/docs/one-eye/logging-operator/crds/) for details.

### Fluentd fails to flush logs to Elasticsearch - Request Entity Too Large

Errors like the following indicate that the HTTP request payload sent by Fluentd exceeds a limit configured either on the Elasticsearch cluster or reverse proxy:

```
error_class=Fluent::Plugin::ElasticsearchOutput::RecoverableRequestFailure error="could not push logs to Elasticsearch cluster [...] [413] Request Content-Length x exceeds the configured limit of y"
```

If you can't raise the limit on the Elasticsearch side, you should make sure that both `bulk_message_request_threshold` and `chunk_limit_size` (Default: 8MB for file-based / 256MB for memory-based buffer) are set to a size below the request limit (leaving some room). For example, given a request limit of 10MB:

```
apiVersion: logging.banzaicloud.io/v1beta1
kind: ClusterOutput
metadata:
  name: elasticsearch
  namespace: cattle-logging-system
spec:
  elasticsearch:
    host: elasticsearch-cluster
    (...)
    bulk_message_request_threshold: 9M
    chunk_limit_size 9M
    (...)
```

### Fluentd stops processing all log events when buffer overflow action is set to block in any output

Some Fluentd example configurations recommend to use `overflow_action block` to deal with output buffer overflow. This option should not be used 
when using the logging operator with multiple outputs as it will in fact [interrupt log processing for all outputs](https://github.com/fluent/fluentd/issues/2327), not just the one in which the buffer overflow occurs.

Instead, the recommended setting for `overflow_action` are `throw_exception` or `drop_oldest_chunk` depending on use case and making sure that
the buffer is properly sized based on average log volume and available throughput towards the logstore.

For use cases that have lower requirements for log retention the `drop_oldest_chunk` option may be a better choice than `throw_exception` as it will not cascade the log flushing problem to the fluent-bit collectors which are even less capable of handling it due to their small, memory-based buffer.

```
apiVersion: logging.banzaicloud.io/v1beta1
kind: ClusterOutput
metadata:
  name: elasticsearch
  namespace: cattle-logging-system
spec:
  elasticsearch:
    host: elasticsearch-cluster
    (...)
    buffer:
      type: file
      overflow_action: throw_exception # drop_oldest_chunk  
    (...)
```

### Fluentd logs spammed with deprecation warnings when using Elasticsearch 7.x

When logging to an Elasticsearch 7.x target, the Fluentd logs may be spammed with warnings like this:

{"type": "deprecation", "timestamp": "2020-10-29T09:12:01,410Z", "level": "WARN", "component": "o.e.d.a.b.BulkRequestParser", "cluster.name": "efk", "node.name": "70dd5c6b94c3", "message": "[types removal] Specifying types in bulk requests is deprecated."}

To solve this, enable the `suppress_type_name` option in the Elasticsearch output spec.

```
apiVersion: logging.banzaicloud.io/v1beta1
kind: ClusterOutput
metadata:
  name: elasticsearch
  namespace: cattle-logging-system
spec:
  elasticsearch:
    host: elasticsearch-cluster
    (...)
    suppress_type_name: true
    (...)
```

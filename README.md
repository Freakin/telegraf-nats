# Testing Telegraf NATS producer/consumer

## Setup 
### NATS
The [NATS](./NATS/) folder contains values for helm chart deployment for a basic 3-node cluster with no auth.  

With Helm configured

`helm install stable/nats --name influx-nats -f NATS/values-production.yaml`

### RBAC

The [RBAC](./RBAC/rbac.yml) file contains the `serviceAccount` and `RoleBinding` with the default `view` ClusterRole.  This allows the `telegraf-consumer` to discover/scrape the metrics from the NATS cluster for additional troubleshooting.  

### NATS Producer
The [telegraf-producer](./telegraf-producer/) folder contains objects to deploy telegraf which will write out to NATS.  It has an influxdb listener so [inch](https://github.com/influxdata/inch) can be used for testing.  Metrics will be written to NATS using internal kubernetes ClusterIP service.

Modify the ingress or service objects depending on your cluster to allow external traffic into the cluster on the appropriate protocol/port.  My testing was done with nginx ingress over HTTPS with no authentication.

Interal metrics are written to stdout, and all other metrics are written to NATS.

### NATS Consumer
The [telegraf-consumer](./telegraf-consumer) folder contains objects to deploy telegraf which consumes from NATS.  


With the default configuration file, the consumed metrics are not emitted and are filtered out.  The `internal_write` metrics will confirm that metrics are being consumed at a high rate from NATS

`telegraf.internal_write,host=telegraf-consumer-0,output=file,version=1.12.3 metrics_dropped=0i,buffer_size=0i,buffer_limit=100000i,metrics_filtered=5825000i,write_time_ns=2073539i,errors=0i,metrics_added=2080i,metrics_written=2080i 1573070550000000000`

#### NATS Cluster Metrics

`inputs.prometheus` is used to discover and scrape NATS cluster metrics, which are written to stdout.  This can be disabled if not desired.



### Influx Receiver
The [telegraf-influx](./telegraf-influx) folder contains objects to deploy telegraf with an influxdb listener, to simulate a remote endpoint that the consumer would write messages to. When output to this instance is enabled, the failure condition can be seen

Internal metrics are written to stdout, received metrics are filtered.


## Testing

### Inch

[Inch](https://github.com/influxdata/inch) is used for testing, to confirm that consuming metrics from NATS works fine while no output is enabled

Inch was used with a basic configuration which emitted a sustained ~35k points/second for testing until terminated

`inch -host https://<telegraf-producer-endpoint>.com -v -m 1000`

### Successful consumption

With no outputs enabled in `telegraf-consumer`, metrics were consumed and filtered at a very high rate

`telegraf.internal_write,host=telegraf-consumer-0,output=file,version=1.12.3 metrics_dropped=0i,buffer_size=0i,buffer_limit=100000i,metrics_filtered=5825000i,write_time_ns=2073539i,errors=0i,metrics_added=2080i,metrics_written=2080i 1573070550000000000`

### Failure Condition

When the output to `telegraf-influx` is enabled using `outputs.influxdb`, the NATS consumer begins failing with Slow Consumer alerts

```
2019-11-06T20:20:21Z I! Started the NATS consumer service, nats: nats://influx-nats-client:4222, subjects: [telegraf], queue: telegraf_consumers
2019-11-06T20:20:23Z E! [inputs.nats_consumer] Error in plugin: nats: slow consumer, messages dropped url:nats://influx-nats-client:4222 id:NDKOFMXQPMRYYIKZOA6HGL34PNJ3SNX3KBTTAXH5MPO3MJKFJQAN4XWM sub:telegraf queue:telegraf_consumers
2019-11-06T20:20:30Z E! [inputs.nats_consumer] Error in plugin: nats: slow consumer, messages dropped url:nats://influx-nats-client:4222 id:NDKOFMXQPMRYYIKZOA6HGL34PNJ3SNX3KBTTAXH5MPO3MJKFJQAN4XWM sub:telegraf queue:telegraf_consumers
```

In addition, the `output=influxdb` metrics show an extremely low rate of metrics being written, indicating that the metrics are not being consumed from NATS appropriately

`telegraf.internal_write,host=telegraf-consumer-0,output=influxdb,version=1.12.3 metrics_filtered=0i,write_time_ns=35997653i,errors=0i,metrics_added=7393i,metrics_written=6393i,metrics_dropped=0i,buffer_size=1000i,buffer_limit=100000i 1573071660000000000`

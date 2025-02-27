# xk6-output-prometheus-remote

> ### ⚠️ Beta version

k6 extension for Prometheus remote-write output.

*Distinguish from [Prometheus remote write **client** extension](https://github.com/grafana/xk6-client-prometheus-remote) :)*

According to [Prometheus API Stability Guarantees](https://prometheus.io/docs/prometheus/latest/stability/) remote write is an **experimental feature**, thus it is unstable and is subject to change. There are many options for remote-write compatible agents, the official list can be found [here](https://prometheus.io/docs/operating/integrations/). The exact details of how metrics will be processed or stored depends on the underlying agent used.

Key points to know:

- remote write format does not contain explicit definition of any metric types while metadata definition is still in flux and can have different implementation depending on the remote-write compatible agent
- remote read is a separate interface and it is much less defined. For example, remote read may not work without precise queries; see [here](https://prometheus.io/docs/prometheus/latest/storage/#remote-storage-integrations) and [here](https://github.com/timescale/promscale/issues/64) for details
- some remote-write compatible agents may support additional formats for remote write, like JSON, but it is not part of official Prometheus remote write specification and therefore absent here

### Usage

To build k6 binary with the Prometheus remote write output extension use:
```
xk6 build --with github.com/grafana/xk6-output-prometheus-remote@latest 
```

Then run new k6 binary with:
```
K6_PROMETHEUS_REMOTE_URL=http://localhost:9090/api/v1/write ./k6 run script.js -o output-prometheus-remote
```

Add TLS and HTTP basic authentication:
```
K6_PROMETHEUS_REMOTE_URL=https://localhost:9090/api/v1/write K6_PROMETHEUS_INSECURE_SKIP_TLS_VERIFY=false K6_CA_CERT_FILE=example/tls.crt K6_PROMETHEUS_USER=foo K6_PROMETHEUS_PASSWORD=bar ./k6 run script.js -o output-prometheus-remote
```

Different remote storage agents are supported with mapping option. The default is Prometheus itself but there is a simpler raw mapping that can be used as a starting point for other remote agents:
```
K6_PROMETHEUS_MAPPING=raw K6_PROMETHEUS_REMOTE_URL=http://localhost:9090/api/v1/write ./k6 run script.js -o output-prometheus-remote
```

Note: Prometheus remote client relies on a snappy library for serialization which can panic on [encode operation](https://github.com/golang/snappy/blob/544b4180ac705b7605231d4a4550a1acb22a19fe/encode.go#L22).

### On sample rate

k6 processes its outputs once per second and that is also a default flush period in this extension. The number of k6 builtin metrics is 26 and they are collected at the rate of 50ms. In practice it means that there will be around 1000-1500 samples on average per each flush period in case of raw mapping. If custom metrics are configured, that estimate will have to be adjusted.

Depending on exact setup, it may be necessary to configure Prometheus and / or remote-write agent to handle the load. For example, see [`queue_config` parameter](https://prometheus.io/docs/practices/remote_write/) of Prometheus.

If remote endpoint responds too slowly or the k6 test run generates too many metrics, extension may start discarding samples in order to continue to adhere to the flush period.

### Prometheus as remote-write agent

To enable remote write in Prometheus 2.x use `--enable-feature=remote-write-receiver` option. See docker-compose samples in `example/`. Options for remote write storage can be found [here](https://prometheus.io/docs/operating/integrations/). 

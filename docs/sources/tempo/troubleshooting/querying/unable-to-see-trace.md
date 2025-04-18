---
title: Unable to find traces
description: Troubleshoot missing traces
weight: 473
aliases:
- ../../operations/troubleshooting/missing-trace/ # https://grafana.com/docs/tempo/<TEMPO_VERSION>/operations/troubleshooting/missing-trace/
- ../../operations/troubleshooting/unable-to-see-trace/ # https://grafana.com/docs/tempo/<TEMPO_VERSION>/operations/troubleshooting/unable-to-see-trace/
- ../unable-to-see-trace/ # htt/docs/tempo/<TEMPO_VERSION>/troubleshooting/unable-to-see-trace/
---

# Unable to find traces

The two main causes of missing traces are:

- Issues in ingestion of the data into Tempo. Spans are either not sent correctly to Tempo or they aren't getting sampled.
- Issues querying for traces that have been received by Tempo.

## Section 1: Diagnose and fix ingestion issues

The first step is to check whether the application spans are actually reaching Tempo.

Add the following flag to the distributor container - [`distributor.log_received_spans.enabled`](https://github.com/grafana/tempo/blob/57da4f3fd5d2966e13a39d27dbed4342af6a857a/modules/distributor/config.go#L55).

This flag enables debug logging of all the traces received by the distributor. These logs can help check if Tempo is receiving any traces at all.

You can also check the following metrics:

- `tempo_distributor_spans_received_total`
- `tempo_ingester_traces_created_total`

The value of both metrics should be greater than `0` within a few minutes of the application spinning up.
You can check both metrics using either:

- The metrics page exposed from Tempo at `http://<tempo-address>:<tempo-http-port>/metrics` or
- In Prometheus, if it's used to scrape metrics.

### Case 1 - `tempo_distributor_spans_received_total` is 0

If the value of `tempo_distributor_spans_received_total` is 0, possible reasons are:

- Use of incorrect protocol/port combination while initializing the tracer in the application.
- Tracing records not getting picked up to send to Tempo by the internal sampler.
- Application is running inside docker and sending traces to an incorrect endpoint.

Receiver specific traffic information can also be obtained using `tempo_receiver_accepted_spans` which has a label for the receiver (protocol used for ingestion. Ex: `jaeger-thrift`).

### Solutions

There are three possible solutions: protocol or port problems, sampling issues, or incorrect endpoints.

To fix protocol or port problems:

- Find out which communication protocol is being used by the application to emit traces. This is unique to every client SDK. For instance: Jaeger Golang Client uses `Thrift Compact over UDP` by default.
- Check the list of supported protocols and their ports and ensure that the correct combination is being used.

To fix sampling issues:

- These issues can be tricky to determine because most SDKs use a probabilistic sampler by default. This may lead to just one in a 1000 records being picked up.
- Check the sampling configuration of the tracer being initialized in the application and make sure it has a high sampling rate.
- Some clients also provide metrics on the number of spans reported from the application, for example `jaeger_tracer_reporter_spans_total`. Check the value of that metric if available and make sure it's greater than zero.
- Another way to diagnose this problem would be to generate lots and lots of traces to see if some records make their way to Tempo.

To fix an incorrect endpoint issue:

- If the application is also running inside docker, make sure the application is sending traces to the correct endpoint (`tempo:<receiver-port>`).

## Case 2 - tempo_ingester_traces_created_total is 0

If the value of `tempo_ingester_traces_created_total` is 0, this can indicate network issues between distributors and ingesters.

Checking the metric `tempo_request_duration_seconds_count{route='/tempopb.Pusher/Push'}` exposed from the ingester which indicates that it's receiving ingestion requests from the distributor.

### Solution

Check logs of distributors for a message like `msg="pusher failed to consume trace data" err="DoBatch: IngesterCount <= 0"`.
This is likely because no ingester is joining the gossip ring, make sure the same gossip ring address is supplied to the distributors and ingesters.

## Diagnose and fix sampling and limits issues

If you are able to query some traces in Tempo but not others, you have come to the right section.

This could happen because of a number of reasons and some have been detailed in this blog post:
[Where did all my spans go? A guide to diagnosing dropped spans in Jaeger distributed tracing](/blog/2020/07/09/where-did-all-my-spans-go-a-guide-to-diagnosing-dropped-spans-in-jaeger-distributed-tracing/).
This is useful if you are using the Jaeger Agent.

If you are using Grafana Alloy, continue reading the following section for metrics to monitor.

### Diagnose the issue

Check if the pipeline is dropping spans. The following metrics on Grafana Alloy help determine this:

- `exporter_send_failed_spans_ratio_total`. The value of this metric should be `0`.
- `receiver_refused_spans_ratio_total`. This value of this metric should be `0`.

If the pipeline isn't reporting any dropped spans, check whether application spans are being dropped by Tempo. The following metrics help determine this:

- `tempo_receiver_refused_spans`. The value of `tempo_receiver_refused_spans` should be `0`.

  If the value of `tempo_receiver_refused_spans` is greater than 0, then the possible reason is the application spans are being dropped due to rate limiting.

### Solutions

- If the pipeline (Grafana Alloy) drops spans, the deployment may need to be scaled up.
- There might also be issues with connectivity to Tempo backend, check Alloy logs and make sure the Tempo endpoint and credentials are correctly configured.
- If Tempo drops spans, this may be due to rate limiting.
  Rate limiting may be appropriate and therefore not an issue. The metric simply explains the cause of the missing spans.
- If you require a higher ingest volume, increase the configuration for the rate limiting by adjusting the `max_traces_per_user` property in the [configured override limits](https://grafana.com/docs/tempo/<TEMPO_VERSION>/configuration/#standard-overrides).

{{< admonition type="note" >}}
Check the [ingestion limits page](https://grafana.com/docs/tempo/<TEMPO_VERSION>/configuration/#overrides) for further information on limits.
{{< /admonition >}}

## Section 3: Diagnose and fix issues with querying traces

If Tempo is correctly ingesting trace spans, then it's time to investigate possible issues with querying the data.

Check the logs of the query-frontend. The query-frontend pod runs with two containers, `query-frontend` and `query`.
Use the following command to view query-frontend logs:

```bash
kubectl logs -f pod/query-frontend-xxxxx -c query-frontend
```

The presence of the following errors in the log may explain issues with querying traces:

- `level=info ts=XXXXXXX caller=frontend.go:63 method=GET traceID=XXXXXXXXX url=/api/traces/XXXXXXXXX duration=5m41.729449877s status=500`
- `no org id`
- `could not dial 10.X.X.X:3200 connection refused`
- `tenant-id not found`

Possible reasons for these errors are:

- The querier isn't connected to the query-frontend. Check the value of the metric `cortex_query_frontend_connected_clients` exposed by the query-frontend.
  It should be > `0`, indicating querier connections with the query-frontend.
- Grafana Tempo data source isn't configured to pass `tenant-id` in the `Authorization` header (multi-tenant deployments only).
- Not connected to Tempo Querier correctly.
- Insufficient permissions.

### Solutions

To fix connection issues:

  - If the queriers aren't connected to the query-frontend, check the following section in the querier configuration and verify the query-frontend address.

    ```yaml
    querier:
      frontend_worker:
        frontend_address: query-frontend-discovery.default.svc.cluster.local:9095
    ```
  - Validate the Grafana data source configuration and debug network issues between Grafana and Tempo.

To fix an insufficient permissions issue:

  - Verify that the querier has the `LIST` and `GET` permissions on the bucket.

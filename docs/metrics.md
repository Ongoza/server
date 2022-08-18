<!--
# Copyright 2018-2022, NVIDIA CORPORATION & AFFILIATES. All rights reserved.
#
# Redistribution and use in source and binary forms, with or without
# modification, are permitted provided that the following conditions
# are met:
#  * Redistributions of source code must retain the above copyright
#    notice, this list of conditions and the following disclaimer.
#  * Redistributions in binary form must reproduce the above copyright
#    notice, this list of conditions and the following disclaimer in the
#    documentation and/or other materials provided with the distribution.
#  * Neither the name of NVIDIA CORPORATION nor the names of its
#    contributors may be used to endorse or promote products derived
#    from this software without specific prior written permission.
#
# THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS ``AS IS'' AND ANY
# EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE
# IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR
# PURPOSE ARE DISCLAIMED.  IN NO EVENT SHALL THE COPYRIGHT OWNER OR
# CONTRIBUTORS BE LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL,
# EXEMPLARY, OR CONSEQUENTIAL DAMAGES (INCLUDING, BUT NOT LIMITED TO,
# PROCUREMENT OF SUBSTITUTE GOODS OR SERVICES; LOSS OF USE, DATA, OR
# PROFITS; OR BUSINESS INTERRUPTION) HOWEVER CAUSED AND ON ANY THEORY
# OF LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY, OR TORT
# (INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE
# OF THIS SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.
-->

# Metrics

Triton provides [Prometheus](https://prometheus.io/) metrics
indicating GPU and request statistics. By default, these metrics are
available at http://localhost:8002/metrics. The metrics are only
available by accessing the endpoint, and are not pushed or published
to any remote server. The metric format is plain text so you can view
them directly, for example:

```
$ curl localhost:8002/metrics
```

The `tritonserver --allow-metrics=false` option can be used to disable
all metric reporting, while the `--allow-gpu-metrics=false` and
`--allow-cpu-metrics=false` can be used to disable just the GPU and CPU
metrics respectively. 

The `--metrics-port` option can be used to select a different port. For now,
Triton reuses http address for metrics endpoint. The option `--http-address`
can be used to bind http and metrics endpoints to the same specific address
when http service is enabled.

**TODO**: 
- `--metrics-interval-ms` mention this
- Update tables to say "Per interval" instead of "Per second" ?

Each section below has a table describing the available metrics.

## Count Metrics

For models that do not support batching, *Request Count*, *Inference
Count* and *Execution Count* will be equal, indicating that each
inference request is executed separately.

For models that support batching, the count metrics can be interpreted
to determine average batch size as *Inference Count* / *Execution
Count*. The count metrics are illustrated by the following examples:

* Client sends a single batch-1 inference request. *Request Count* =
  1, *Inference Count* = 1, *Execution Count* = 1.

* Client sends a single batch-8 inference request. *Request Count* =
  1, *Inference Count* = 8, *Execution Count* = 1.

* Client sends 2 requests: batch-1 and batch-8. Dynamic batcher is not
  enabled for the model. *Request Count* = 2, *Inference Count* = 9,
  *Execution Count* = 2.

* Client sends 2 requests: batch-1 and batch-1. Dynamic batcher is
  enabled for the model and the 2 requests are dynamically batched by
  the server. *Request Count* = 2, *Inference Count* = 2, *Execution
  Count* = 1.

* Client sends 2 requests: batch-1 and batch-8. Dynamic batcher is
  enabled for the model and the 2 requests are dynamically batched by
  the server. *Request Count* = 2, *Inference Count* = 9, *Execution
  Count* = 1.

|Category      |Metric          |Metric Name |Description                            |Granularity|Frequency    |
|--------------|----------------|------------|---------------------------|-----------|-------------|
|Count         |Success Count   |`nv_inference_request_success` |Number of successful inference requests received by Triton (each request is counted as 1, even if the request contains a batch) |Per model  |Per request  |
|              |Failure Count   |`nv_inference_request_failure` |Number of failed inference requests received by Triton (each request is counted as 1, even if the request contains a batch) |Per model  |Per request  |
|              |Inference Count |`nv_inference_count` |Number of inferences performed (a batch of "n" is counted as "n" inferences, does not include cached requests)|Per model|Per request|
|              |Execution Count |`nv_inference_exec_count` |Number of inference batch executions (see [Count Metrics](#count-metrics), does not include cached requests)|Per model|Per request|
|Latency       |Request Time    |`nv_inference_request_duration_us` |Cumulative end-to-end inference request handling time (includes cached requests) |Per model  |Per request  |
|              |Queue Time      |`nv_inference_queue_duration_us` |Cumulative time requests spend waiting in the scheduling queue (includes cached requests) |Per model  |Per request  |
|              |Compute Input Time|`nv_inference_compute_input_duration_us` |Cumulative time requests spend processing inference inputs (in the framework backend, does not include cached requests)     |Per model  |Per request  |
|              |Compute Time    |`nv_inference_compute_infer_duration_us` |Cumulative time requests spend executing the inference model (in the framework backend, does not include cached requests)     |Per model  |Per request  |
|              |Compute Output Time|`nv_inference_compute_output_duration_us` |Cumulative time requests spend processing inference outputs (in the framework backend, does not include cached requests)     |Per model  |Per request  |

## Response Cache

Compute latency metrics in the table above are calculated for the
time spent in model inference backends. If the response cache is enabled for a
given model (see [Response Cache](https://github.com/triton-inference-server/server/blob/main/docs/response_cache.md)
docs for more info), total inference times may be affected by response cache
lookup times.

On cache hits, "Cache Hit Lookup Time" indicates the time spent looking up the
response, and "Compute Input Time" /  "Compute Time" / "Compute Output Time"
are not recorded.

On cache misses, "Cache Miss Lookup Time" indicates the time spent looking up
the request hash and "Cache Miss Insertion Time" indicates the time spent
inserting the computed output tensor data into the cache. Otherwise, "Compute
Input Time" /  "Compute Time" / "Compute Output Time" will be recorded as usual.

|Category      |Metric          |Metric Name |Description                            |Granularity|Frequency    |
|--------------|----------------|------------|---------------------------|-----------|-------------|
|Count         |Total Cache Entry Count |`nv_cache_num_entries` |Total number of responses stored in response cache across all models |Server-wide |Per second |
|              |Total Cache Lookup Count |`nv_cache_num_lookups` |Total number of response cache lookups done by Triton across all models |Server-wide |Per second |
|              |Total Cache Hit Count |`nv_cache_num_hits` |Total number of response cache hits across all models |Server-wide |Per second |
|              |Total Cache Miss Count |`nv_cache_num_misses` |Total number of response cache misses across all models |Server-wide |Per second |
|              |Total Cache Eviction Count |`nv_cache_num_evictions` |Total number of response cache evictions across all models |Server-wide |Per second |
|              |Cache Hit Count |`nv_cache_num_hits_per_model` |Number of response cache hits per model |Per model |Per request |
|              |Cache Miss Count |`nv_cache_num_misses_per_model` |Number of response cache misses per model |Per model |Per request |
|Latency       |Total Cache Lookup Time |`nv_cache_lookup_duration` |Cumulative time requests spend checking for a cached response across all models (microseconds) |Server-wide |Per second |
|              |Total Cache Insertion Time |`nv_cache_insertion_duration` |Cumulative time requests spend inserting a response into the cache across all models (microseconds) |Server-wide |Per second |
|              |Cache Hit Lookup Time |`nv_cache_hit_lookup_duration_per_model` |Cumulative time requests spend retrieving a cached response per model on cache hits (microseconds) |Per model |Per request |
|              |Cache Miss Lookup Time |`nv_cache_miss_lookup_duration_per_model`| Cumulative time requests spend looking up a request hash on a cache miss (microseconds) |Per model |Per request |
|              |Cache Miss Insertion Time |`nv_cache_miss_insertion_duration_per_model` |Cumulative time requests spend inserting responses into the cache on a cache miss (microseconds) |Per model |Per request |
|Utilization   |Total Cache Utilization |`nv_cache_util` |Total Response Cache utilization rate (0.0 - 1.0) |Server-wide |Per second |


## GPU Metrics

TODO:
- Optionally toggled with `TRITON_ENABLE_METRICS_GPU`, default True
- `--allow-gpu-metrics`
- Details: DCGM
- Known issue with separate DCGM agent

|Category      |Metric          |Metric Name |Description                            |Granularity|Frequency    |
|--------------|----------------|------------|---------------------------|-----------|-------------|
|GPU Utilization |Power Usage   |`nv_gpu_power_usage` |GPU instantaneous power                |Per GPU    |Per second   |
|              |Power Limit     |`nv_gpu_power_limit` |Maximum GPU power limit                |Per GPU    |Per second   |
|              |Energy Consumption|`nv_energy_consumption` |GPU energy consumption in joules since Triton started|Per GPU|Per second|
|              |GPU Utilization |`nv_gpu_utilization` |GPU utilization rate (0.0 - 1.0)       |Per GPU    |Per second   |
|GPU Memory    |GPU Total Memory|`nv_gpu_memory_total_bytes` |Total GPU memory, in bytes             |Per GPU    |Per second   |
|              |GPU Used Memory |`nv_gpu_memory_used_bytes` |Used GPU memory, in bytes              |Per GPU    |Per second   |


## CPU Metrics

TODO:
- Linux-only
- Optionally toggled with `TRITON_ENABLE_METRICS_CPU`, default True?
- `--allow-cpu-metrics`
- Details: `/proc/stat` ?

|Category      |Metric          |Metric Name |Description                            |Granularity|Frequency    |
|--------------|----------------|------------|---------------------------|-----------|-------------|
|CPU Utilization | TBD | `TBD` | TBD | Per CPU | Per second |
|CPU Memory      | TBD | `TBD` | TBD | Per CPU | Per second |

## Custom Metrics

Triton exposes a C API to allow users and backends to register and collect
custom metrics with the existing Triton metrics endpoint. The user takes the
ownership of the custom metrics created through the APIs and must manage their
lifetime following the API documentation.

The 
[identity_backend](https://github.com/triton-inference-server/identity_backend/blob/main/README.md#custom-metric-example)
demonstrates a practical example of adding a custom metric to a backend.

Further documentation can be found in the `TRITONSERVER_MetricFamily*` and
`TRITONSERVER_Metric*` API annotations in
[tritonserver.h](https://github.com/triton-inference-server/core/blob/main/include/triton/core/tritonserver.h).

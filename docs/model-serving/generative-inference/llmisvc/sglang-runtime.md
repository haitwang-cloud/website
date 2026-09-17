---
sidebar_label: "SGLang Runtime"
sidebar_position: 7
title: "Run LLMInferenceService with the SGLang Runtime"
description: End-to-end guide for serving a Hugging Face model with KServe LLMInferenceService and the SGLang runtime
---

# Run LLMInferenceService with the SGLang Runtime

This guide deploys an [SGLang](https://github.com/sgl-project/sglang) model server with KServe `LLMInferenceService` and verifies its OpenAI-compatible `/v1/completions` endpoint. The primary example uses one GPU and tests the workload directly. Managed Gateway API routing and scheduling are covered as an optional extension.

## Supported deployment

KServe provides the `kserve-llm-sglang` `ClusterServingRuntime` for single-node, non-disaggregated `LLMInferenceService` deployments.

The runtime:

- Must be selected explicitly with `spec.runtime: kserve-llm-sglang`.
- Starts `python3 -m sglang.launch_server` on port `8000`.
- Mounts the model downloaded by KServe at `/mnt/models`.
- Uses `lmsysorg/sglang:v0.5.14` by default.
- Maps `spec.parallelism.tensor` to the SGLang `--tp` argument.
- Adds `--trust-remote-code` when `spec.trustRemoteCode` is enabled.

The built-in SGLang integration does not provide multi-node or disaggregated prefill-decode templates. Use the default llm-d/vLLM configuration for those deployment patterns.

## Prerequisites

Before you begin, make sure that:

- `kubectl` is configured for a cluster with `LLMInferenceService` installed.
- The installation includes the built-in `kserve-llm-sglang` runtime.
- At least one node advertises an allocatable `nvidia.com/gpu`.
- Workload pods can access Hugging Face to download `facebook/opt-125m`.

Verify the CRD, runtime, and GPU capacity:

```bash
kubectl get crd llminferenceservices.serving.kserve.io
kubectl get clusterservingruntime kserve-llm-sglang
kubectl get nodes \
  -o jsonpath='{range .items[*]}{.metadata.name}{" gpu="}{.status.allocatable.nvidia\.com/gpu}{"\n"}{end}'
```

See [LLMInferenceService dependencies](./llmisvc-dependencies.md) if the CRD or runtime is not installed.

## Deploy a GPU workload

Apply a single-node `LLMInferenceService`:

```bash
kubectl apply -f - <<'EOF'
apiVersion: serving.kserve.io/v1alpha2
kind: LLMInferenceService
metadata:
  name: sglang-opt-125m
spec:
  runtime: kserve-llm-sglang
  model:
    uri: hf://facebook/opt-125m
    name: facebook/opt-125m
  replicas: 1
  template:
    containers:
      - name: main
        resources:
          limits:
            cpu: "4"
            memory: 16Gi
            nvidia.com/gpu: "1"
          requests:
            cpu: "1"
            memory: 8Gi
            nvidia.com/gpu: "1"
EOF
```

KServe downloads the model with the storage initializer, mounts it at `/mnt/models`, and starts SGLang on port `8000`. The model download and initial server startup can take several minutes.

## Verify readiness

Wait for the service:

```bash
kubectl wait --for=condition=Ready \
  llminferenceservice/sglang-opt-125m \
  --timeout=1800s
```

Inspect the status conditions:

```bash
kubectl get llminferenceservice sglang-opt-125m \
  -o jsonpath='{range .status.conditions[*]}{.type}={.status} reason={.reason}{"\n"}{end}'
```

A ready single-node workload reports these conditions as `True`:

```text
PresetsCombined=True
MainWorkloadReady=True
WorkloadsReady=True
Ready=True
```

Confirm that the workload pod is running and has a GPU:

```bash
kubectl get pod \
  -l app.kubernetes.io/name=sglang-opt-125m,app.kubernetes.io/component=llminferenceservice-workload \
  -o jsonpath='{range .items[*]}{.metadata.name}{" phase="}{.status.phase}{" node="}{.spec.nodeName}{" gpu="}{.spec.containers[0].resources.limits.nvidia\.com/gpu}{"\n"}{end}'
```

## Test inference

In one terminal, forward a local port to the SGLang workload:

```bash
kubectl port-forward \
  service/sglang-opt-125m-kserve-workload-svc \
  18000:8000
```

In another terminal, send an OpenAI-compatible completion request:

```bash
curl -sS http://127.0.0.1:18000/v1/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "facebook/opt-125m",
    "prompt": "What is Kubernetes?",
    "max_tokens": 16
  }'
```

A successful response has `object: "text_completion"` and contains generated text in `choices`:

```json
{
  "object": "text_completion",
  "model": "facebook/opt-125m",
  "choices": [
    {
      "text": " ..."
    }
  ]
}
```

This request verifies the KServe storage initialization, SGLang runtime configuration, workload, Service, and inference endpoint without requiring a Gateway provider.

## Optional: Enable managed routing and scheduling

To expose the service through Gateway API and use the managed scheduler, first install the [LLMInferenceService routing dependencies](./llmisvc-dependencies.md). The cluster must include Gateway API, Gateway API Inference Extension, a compatible Gateway provider, and an `InferencePool` extension.

Enable the managed components:

```bash
kubectl patch llminferenceservice sglang-opt-125m \
  --type=merge \
  --patch '{
    "spec": {
      "router": {
        "scheduler": {},
        "route": {},
        "gateway": {}
      }
    }
  }'
```

Wait for the service to reconcile, then inspect its conditions:

```bash
kubectl wait --for=condition=Ready \
  llminferenceservice/sglang-opt-125m \
  --timeout=10m

kubectl get llminferenceservice sglang-opt-125m \
  -o jsonpath='{range .status.conditions[*]}{.type}={.status} reason={.reason}{"\n"}{end}'
```

In addition to the workload conditions, a healthy managed route reports:

```text
InferencePoolReady=True
SchedulerWorkloadReady=True
HTTPRoutesReady=True
RouterReady=True
Ready=True
```

Use the URL reported by the service instead of assuming a Gateway address or route prefix:

```bash
SERVICE_URL=$(kubectl get llminferenceservice sglang-opt-125m \
  -o jsonpath='{.status.url}')

curl -sS "${SERVICE_URL}/v1/completions" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "facebook/opt-125m",
    "prompt": "What is Kubernetes?",
    "max_tokens": 16
  }'
```

## Optional: Run on CPU

For development environments without a GPU, override the default runtime image with the SGLang CPU image:

```bash
kubectl apply -f - <<'EOF'
apiVersion: serving.kserve.io/v1alpha2
kind: LLMInferenceService
metadata:
  name: sglang-opt-125m-cpu
spec:
  runtime: kserve-llm-sglang
  model:
    uri: hf://facebook/opt-125m
    name: facebook/opt-125m
  replicas: 1
  template:
    containers:
      - name: main
        image: lmsysorg/sglang:v0.5.13-xeon
        resources:
          limits:
            cpu: "4"
            memory: 16Gi
          requests:
            cpu: "1"
            memory: 8Gi
EOF
```

Wait for `llminferenceservice/sglang-opt-125m-cpu`, then repeat the direct inference test with `service/sglang-opt-125m-cpu-kserve-workload-svc`.

## Troubleshooting

### Runtime is not found

If the service reports that `kserve-llm-sglang` does not exist, verify that the runtime was installed:

```bash
kubectl get clusterservingruntime kserve-llm-sglang -o yaml
```

The SGLang runtime is not selected automatically. Make sure the service includes `spec.runtime: kserve-llm-sglang`.

### Model pod is stuck during initialization

Find the workload pod and check the storage initializer:

```bash
POD=$(kubectl get pod \
  -l app.kubernetes.io/name=sglang-opt-125m,app.kubernetes.io/component=llminferenceservice-workload \
  -o jsonpath='{.items[0].metadata.name}')

kubectl logs "${POD}" -c storage-initializer
```

The initializer must be able to access Hugging Face. Private models also require a Hugging Face token configured for model storage.

### Workload remains pending

Inspect scheduling events and GPU capacity:

```bash
kubectl describe pod \
  -l app.kubernetes.io/name=sglang-opt-125m,app.kubernetes.io/component=llminferenceservice-workload

kubectl get nodes \
  -o jsonpath='{range .items[*]}{.metadata.name}{" gpu="}{.status.allocatable.nvidia\.com/gpu}{"\n"}{end}'
```

The GPU example requires one allocatable `nvidia.com/gpu`.

### Managed route is not ready

Inspect the generated route:

```bash
kubectl get httproute sglang-opt-125m-kserve-route \
  -o jsonpath='{range .status.parents[*].conditions[*]}{.type}={.status} reason={.reason} message={.message}{"\n"}{end}'
```

`ResolvedRefs=False` with `InvalidKind` for `inference.networking.k8s.io/InferencePool` means the Gateway provider is not configured for the Inference Extension. Verify the routing dependencies and restart the Gateway controller after enabling its `InferencePool` extension.

## Clean up

Delete the example when you finish:

```bash
kubectl delete llminferenceservice sglang-opt-125m
```

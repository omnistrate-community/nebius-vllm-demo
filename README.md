# Nebius vLLM Omnistrate Specs

This repository contains two `ServicePlanSpec` variants for running [vLLM](https://github.com/vllm-project/vllm) on Nebius through Omnistrate.

Both specs are designed to work out of the box as Nebius-hosted plans with:

- a published OCI Helm chart at `oci://ghcr.io/omnistrate/vllm`
- the `vllm/vllm-openai:latest` image
- local persistent storage for Hugging Face model cache and weights
- **Customer-visible observability** through `features.CUSTOMER.metrics`
  - vLLM serving Grafana dashboards managed by Omnistrate
    - Traffic, latency, cache efficiency, and token throughput
  - NVIDIA GPU Grafana dashboards backed by `dcgm-exporter` and managed by Omnistrate
    - GPU inventory, utilization, framebuffer usage, PCIe throughput, thermals, power, and XID errors
- **Hosted Nebius deployment** via `deployment.hostedDeployment.NebiusTenantId`
- **External DNS wiring** via `$sys.network.externalClusterEndpoint`
- **Public inference endpoint** using `endpointConfiguration` plus a `LoadBalancer` Helm service
- **vLLM OpenAI-compatible API** on port `8000`
- **Local model storage** on a PVC mounted at `/data`

## Files

| File | Use case | GPU shape | Notes |
| --- | --- | --- | --- |
| `spec.yaml` | Single-node, non-cluster Nebius GPU deployment | `1 x H100` | Simplest starting point for bringing up vLLM on Nebius |
| `spec-gpu-cluster.yaml` | Nebius GPU cluster deployment | `8 x H100` | Keeps `GpuClusterID` support and enables vLLM tensor parallelism |

## `spec.yaml`

`spec.yaml` is the simpler deployment path.

### Compute profile

- Nebius preset: `1gpu-16vcpu-200gb`
- Platform: `gpu-h200-sxm`
- GPU request/limit: `1`

## `spec-gpu-cluster.yaml`

`spec-gpu-cluster.yaml` keeps the same application, endpoint, and observability model, but is sized for Nebius GPU-cluster-backed deployment.

### Compute profile

- Nebius preset: `8gpu-128vcpu-1600gb`
- Platform: `gpu-h200-sxm`
- `configurationOverrides.GpuClusterID: <INSERT GPU CLUSTER ID>`
- GPU request/limit: `8`
- one vLLM replica sized for 8 GPUs
- Nebius GPU cluster placement through `GpuClusterID`
- `--tensor-parallel-size 8` in the vLLM command

## Key Runtime Defaults

Both files currently ship with these important runtime defaults:

- model: `Jackrong/Qwen3.5-27B-Claude-4.6-Opus-Reasoning-Distilled`
- tokenizer override: `Qwen/Qwen3.5-27B`
- served model name: `qwen3.5-27b-claude-4.6-opus-reasoning-distilled`
- dtype: `auto`
- image: `vllm/vllm-openai:latest`
- chart version: `0.0.6`
- chart repo: `oci://ghcr.io/omnistrate/vllm`

Neither spec sets a `runtimeClassName` by default. That is intentional so the same specs work on Nebius clusters that expose GPUs through the default container runtime and do not publish a `RuntimeClass` object such as `nebius-nvidia`.

## Observability Included

Out of the box, the Omnistrate dashboard experience includes:

- **vLLM Serving**
  - running and waiting requests
  - requests per second
  - prompt and generation token throughput
  - TTFT, TPOT, queue time, and end-to-end latency
  - KV cache usage
  - prefix cache hit rate

- **NVIDIA GPU**
  - GPU count and inventory
  - GPU utilization
  - memory copy utilization
  - framebuffer used and utilization percent
  - PCIe RX/TX throughput
  - power draw
  - temperature
  - SM and memory clocks
  - XID and PCIe replay error visibility

## Storage Behavior

Both specs are configured to use local persistent volume storage for model data under `/data`.

That means:

- Hugging Face artifacts persist on the attached volume

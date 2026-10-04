# NVIDIA RAG Blueprint — NIM-LLM Troubleshooting & `values-mig-h100.yaml` Changes

## 1. Purpose

This document records the troubleshooting performed while upgrading the NVIDIA RAG Blueprint Helm deployment and getting the `nim-llm` Nemotron 3 Super 120B NIM running successfully.

It covers:

- Helm chart upgrade from the existing deployment to RAG Blueprint v2.6.0 configuration with Nemotron NIM 2.0.9.
- NIMCache immutability handling.
- NGC credential troubleshooting.
- H100 GPU and MIG resource validation.
- `nim-llm` GPU visibility validation.
- vLLM startup/Inductor troubleshooting.
- Changes made to `mig-slicing/values-mig-h100.yaml`.
- Final working configuration and recommended troubleshooting commands.

---

## 2. Environment

| Component | Configuration |
|---|---|
| Kubernetes distribution | RKE2 |
| Kubernetes namespace | `rag` |
| Helm release | `rag` |
| Helm chart | `nvidia-blueprint-rag-v2.6.0` |
| Node | `hclaif` |
| Physical GPUs | 4 × NVIDIA H100 NVL |
| GPU memory | ~95,830 MiB per H100 |
| NVIDIA driver | 580.178.04 |
| Host CUDA | 13.0 reported by node tooling |
| NIM CUDA environment | CUDA 13.1 reported inside NIM container |
| LLM | `nvidia/nemotron-3-super-120b-a12b` |
| LLM NIM version | `2.0.9` |
| Inference engine | vLLM |
| Tensor parallelism | 2 |
| LLM GPU allocation | 2 × full H100 |
| MIG resources | `mig-1g.24gb`, `mig-3g.47gb` |

---

## 3. Intended GPU Allocation

The target configuration was:

| Workload | GPU resource |
|---|---|
| Nemotron 3 Super 120B (`nim-llm`) | 2 × full `nvidia.com/gpu` |
| OCR | 1 × `nvidia.com/mig-3g.47gb` |
| VLM Embedding | 1 × `nvidia.com/mig-3g.47gb` |
| Graphic elements | 1 × `nvidia.com/mig-1g.24gb` |
| Page elements | 1 × `nvidia.com/mig-1g.24gb` |
| Table structure | 1 × `nvidia.com/mig-1g.24gb` |
| Reranking | 1 × `nvidia.com/mig-1g.24gb` |

The node ultimately reported:

```text
nvidia.com/gpu:          6
nvidia.com/mig-1g.24gb: 4
nvidia.com/mig-3g.47gb: 2
```

The six `nvidia.com/gpu` resources represent the full-GPU resource view exposed by the configured GPU sharing/MIG setup; the workload allocation was validated separately using `kubectl-view_allocations`.

---

# 4. Troubleshooting Timeline

## 4.1 Initial Helm upgrade failure — immutable NIMCache

The existing `nim-llm-cache` was already present.

Its existing model puller referenced an older NIM image:

```text
nvcr.io/nim/nvidia/nemotron-3-super-120b-a12b:1.8.0
```

The live NIMCache had subsequently been associated with:

```text
nvcr.io/nim/nvidia/nemotron-3-super-120b-a12b:2.0.9
```

When Helm attempted to reconcile the resource, the NIM Operator rejected the update because:

```text
spec.source.ngc.modelPuller
```

is immutable.

Typical error:

```text
NIMCache.apps.nvidia.com "nim-llm-cache" is invalid:
spec.source.ngc.modelPuller: Invalid value:
modelPuller is an immutable field.
Please create a new NIMCache resource instead when you want to
change this container.
```

### Resolution

Only the NIMCache object was removed:

```bash
kubectl delete nimcache nim-llm-cache -n rag
```

The cache PVC was preserved:

```text
nim-llm-cache-pvc
```

The RAG Helm release and the cache PVC were not deleted.

---

# 5. Helm Template Rendering Issue

An initial:

```bash
helm template
```

did not show `NIMCache` or `NIMService` resources.

The chart does contain the relevant templates:

```text
templates/embedding-nim.yaml
templates/llm-nim.yaml
templates/reranking-nim.yaml
templates/vlm-captioning-nim.yaml
templates/vlm-embed-nim.yaml
templates/vlm-nim.yaml
templates/vlm-reranker-nim.yaml
```

The important discovery was that Helm's template rendering needed the NVIDIA NIM Operator API version to be declared explicitly during offline rendering:

```bash
--api-versions apps.nvidia.com/v1alpha1
```

Correct render command:

```bash
helm template rag \
  https://helm.ngc.nvidia.com/nvidia/blueprint/charts/nvidia-blueprint-rag-v2.6.0.tgz \
  -n rag \
  --username '$oauthtoken' \
  --password "$SUSE_AIF_NVIDIA_API_KEY" \
  -f mig-slicing/values-mig-h100.yaml \
  --api-versions apps.nvidia.com/v1alpha1 \
  > /tmp/rag-rendered.yaml
```

After this, the rendered output contained:

```text
NIMCache
NIMService
```

resources as expected.

---

# 6. NIM Version Change

The chart default for `nim-llm` was:

```yaml
image:
  repository: nvcr.io/nim/nvidia/nemotron-3-super-120b-a12b
  tag: "1.8.0"
```

The required version was changed to:

```yaml
nimOperator:
  nim-llm:
    image:
      tag: "2.0.9"
```

The final rendered resource confirmed:

```yaml
image:
  repository: nvcr.io/nim/nvidia/nemotron-3-super-120b-a12b
  tag: 2.0.9
```

---

# 7. Reranker Resource Correction

The desired architecture uses one MIG slice for reranking rather than one full H100.

The active chart key was identified as:

```yaml
nvidia-nim-llama-nemotron-rerank-1b-v2:
```

The resource configuration was changed to:

```yaml
nvidia-nim-llama-nemotron-rerank-1b-v2:
  resources:
    limits:
      nvidia.com/gpu: "0"
      nvidia.com/mig-1g.24gb: 1
    requests:
      nvidia.com/gpu: "0"
      nvidia.com/mig-1g.24gb: 1
```

This resulted in the following Kubernetes allocation:

```text
nemotron-ranking-ms -> 1 × mig-1g.24gb
```

The reranker therefore no longer consumes one full `nvidia.com/gpu`.

---

# 8. LLM Resource Configuration

The `nim-llm` service was configured for two full H100 GPUs:

```yaml
nim-llm:
  image:
    tag: "2.0.9"

  resources:
    limits:
      nvidia.com/gpu: 2
    requests:
      nvidia.com/gpu: 2
```

The NIM model configuration remains:

```yaml
model:
  engine: vllm
  precision: "fp8"
  tensorParallelism: "2"
```

This means:

- TP worker 0 uses one H100.
- TP worker 1 uses one H100.
- Tensor parallelism is 2.

---

# 9. GPU Visibility Troubleshooting

At one point, the container environment showed:

```text
NVIDIA_VISIBLE_DEVICES=void
```

This looked suspicious.

However, after the NIM was recreated and running correctly, the container showed:

```text
GPU 0: NVIDIA H100 NVL
GPU 1: NVIDIA H100 NVL
```

and:

```text
GPU 0 -> VLLM::Worker_TP0
GPU 1 -> VLLM::Worker_TP1
```

with approximately:

```text
63002 MiB
```

used on each GPU during model initialization.

Therefore:

- Kubernetes was allocating 2 GPUs correctly.
- Both GPUs were visible to vLLM.
- `NVIDIA_VISIBLE_DEVICES=void` was not the final root cause.
- No manual modification of `NVIDIA_VISIBLE_DEVICES` was required.

Validation command:

```bash
kubectl exec -n rag <nim-llm-pod> -- nvidia-smi -L
```

Expected:

```text
GPU 0: NVIDIA H100 NVL
GPU 1: NVIDIA H100 NVL
```

---

# 10. NGC Credential Problem

The first rebuilt NIMCache remained `Pending`.

NIM Operator logs showed:

```text
401 Unauthorized
```

when trying to inspect/pull:

```text
nvcr.io/nim/nvidia/nemotron-3-super-120b-a12b:2.0.9
```

The critical discovery was that the `ngc-api` Secret existed but contained empty values:

```yaml
NGC_API_KEY: ""
NGC_CLI_API_KEY: ""
NVIDIA_API_KEY: ""
```

This directly explained the registry authentication failure.

### Credential validation

Use:

```bash
kubectl get secret ngc-api -n rag
kubectl get secret ngc-secret -n rag
```

Check that the API key field is populated without printing the key:

```bash
kubectl get secret ngc-api -n rag \
  -o jsonpath='{.data.NGC_API_KEY}' | base64 -d | wc -c
```

A result of `0` means the secret is empty.

### Credential refresh

The Helm upgrade was performed with:

```bash
--set-string imagePullSecret.password="$SUSE_AIF_NVIDIA_API_KEY"
--set-string ngcApiSecret.password="$SUSE_AIF_NVIDIA_API_KEY"
```

This populated the credentials used by the NIM deployment.

---

# 11. NIMCache Recreation

After the credentials were corrected, the failed `nim-llm-cache` object was recreated.

Important:

Do not delete:

```text
nim-llm-cache-pvc
```

The PVC was retained and remained bound.

The NIMCache subsequently became usable and the NIMService was able to start the model.

---

# 12. vLLM Startup Failure After Successful Model Download

After the cache problem was resolved, the NIM container began loading the Nemotron model.

Evidence:

```text
Loading safetensors checkpoint shards: 100% Completed | 26/26
```

and:

```text
Model loading took 57.57 GiB memory
```

The failure occurred after model loading during vLLM/PyTorch Inductor compilation.

The error ended with:

```text
torch._inductor.exc.InductorError: AssertionError
```

and:

```text
RuntimeError: Worker failed with error 'AssertionError: '
```

The stack was inside:

```text
torch._inductor
vllm
cuda graph compilation
```

The problem was traced to the way the `env` list had been overridden in the custom values file.

---

# 13. Important `env` Configuration Change

An earlier custom values section replaced the chart's default NIM environment list with only:

```yaml
env:
  - name: NIM_PASSTHROUGH_ARGS
    value: "--max-num-seqs 384"
```

This caused the rendered chart to contain several `null` entries and omitted the default NIM runtime configuration.

The correct configuration restores the normal NIM environment entries and adds:

```yaml
NIM_PASSTHROUGH_ARGS=--max-num-seqs 384
```

The final rendered `nim-llm` environment became:

```yaml
env:
  - name: NIM_HTTP_API_PORT
    value: "8000"

  - name: NIM_TRITON_LOG_VERBOSE
    value: "1"

  - name: NIM_SERVED_MODEL_NAME
    value: "nvidia/nemotron-3-super-120b-a12b"

  - name: NIM_ENABLE_CHUNKED_PREFILL
    value: "1"

  - name: NCCL_NVLS_ENABLE
    value: "0"

  - name: VLLM_USE_FLASHINFER_MOE_FP8
    value: "0"

  - name: NIM_PASSTHROUGH_ARGS
    value: "--max-num-seqs 384"
```

The final rendered manifest contained no `null` entries.

---

# 14. Recommended `nim-llm` Section

The resulting `nim-llm` configuration is:

```yaml
nimOperator:
  nim-llm:
    image:
      tag: "2.0.9"

    resources:
      limits:
        nvidia.com/gpu: 2
      requests:
        nvidia.com/gpu: 2

    storage:
      pvc:
        storageClass: ""

    env:
      - name: NIM_HTTP_API_PORT
        value: "8000"
      - name: NIM_TRITON_LOG_VERBOSE
        value: "1"
      - name: NIM_SERVED_MODEL_NAME
        value: "nvidia/nemotron-3-super-120b-a12b"
      - name: NIM_ENABLE_CHUNKED_PREFILL
        value: "1"
      - name: NCCL_NVLS_ENABLE
        value: "0"
      - name: VLLM_USE_FLASHINFER_MOE_FP8
        value: "0"
      - name: NIM_PASSTHROUGH_ARGS
        value: "--max-num-seqs 384"
```

---

# 15. Final MIG Configuration

The custom values file contains the following workload placement.

## OCR

```yaml
ocr:
  resources:
    limits:
      nvidia.com/gpu: "0"
      nvidia.com/mig-3g.47gb: 1
    requests:
      nvidia.com/gpu: "0"
      nvidia.com/mig-3g.47gb: 1
```

## Graphic Elements

```yaml
graphic_elements:
  resources:
    limits:
      nvidia.com/gpu: "0"
      nvidia.com/mig-1g.24gb: 1
    requests:
      nvidia.com/gpu: "0"
      nvidia.com/mig-1g.24gb: 1
```

## Page Elements

```yaml
page_elements:
  resources:
    limits:
      nvidia.com/gpu: "0"
      nvidia.com/mig-1g.24gb: 1
    requests:
      nvidia.com/gpu: "0"
      nvidia.com/mig-1g.24gb: 1
```

## Table Structure

```yaml
table_structure:
  resources:
    limits:
      nvidia.com/gpu: "0"
      nvidia.com/mig-1g.24gb: 1
    requests:
      nvidia.com/gpu: "0"
      nvidia.com/mig-1g.24gb: 1
```

## VLM Embedding

```yaml
nvidia-nim-llama-nemotron-embed-vl-1b-v2:
  resources:
    limits:
      nvidia.com/gpu: "0"
      nvidia.com/mig-3g.47gb: 1
    requests:
      nvidia.com/gpu: "0"
      nvidia.com/mig-3g.47gb: 1
```

## Reranking

```yaml
nvidia-nim-llama-nemotron-rerank-1b-v2:
  resources:
    limits:
      nvidia.com/gpu: "0"
      nvidia.com/mig-1g.24gb: 1
    requests:
      nvidia.com/gpu: "0"
      nvidia.com/mig-1g.24gb: 1
```

---

# 16. Final Validation Commands

## Helm release

```bash
helm status rag -n rag
```

## NIM resources

```bash
kubectl get nimcache,nimservice -n rag
```

Expected for LLM:

```text
nim-llm-cache   Ready
nim-llm         Ready
```

## Pods

```bash
kubectl get pods -n rag
```

## GPU allocation

```bash
kubectl-view_allocations
```

## LLM GPU visibility

```bash
kubectl exec -n rag <nim-llm-pod> -- nvidia-smi -L
```

Expected:

```text
GPU 0: NVIDIA H100 NVL
GPU 1: NVIDIA H100 NVL
```

## Detailed GPU usage

```bash
kubectl exec -n rag <nim-llm-pod> -- nvidia-smi
```

## NIM environment

```bash
kubectl exec -n rag <nim-llm-pod> -- \
  printenv | grep -E 'NVIDIA_VISIBLE_DEVICES|CUDA_VISIBLE_DEVICES|NIM_'
```

## NIM logs

```bash
kubectl logs -n rag <nim-llm-pod> --tail=200
```

For a restarting pod:

```bash
kubectl logs -n rag <nim-llm-pod> --previous --tail=200
```

---

# 17. Safe Helm Upgrade Procedure

Use template rendering first:

```bash
helm template rag \
  https://helm.ngc.nvidia.com/nvidia/blueprint/charts/nvidia-blueprint-rag-v2.6.0.tgz \
  -n rag \
  --username '$oauthtoken' \
  --password "$SUSE_AIF_NVIDIA_API_KEY" \
  -f mig-slicing/values-mig-h100.yaml \
  --api-versions apps.nvidia.com/v1alpha1 \
  > /tmp/rag-rendered.yaml
```

Check NIM resources:

```bash
grep -nE '^kind: (NIMService|NIMCache)' /tmp/rag-rendered.yaml
```

Check LLM version:

```bash
grep -n -A8 -B5 \
  'repository: nvcr.io/nim/nvidia/nemotron-3-super-120b-a12b' \
  /tmp/rag-rendered.yaml
```

Check the generated environment:

```bash
grep -n -A25 -B3 'name: nim-llm' /tmp/rag-rendered.yaml
```

Then upgrade:

```bash
helm upgrade rag \
  https://helm.ngc.nvidia.com/nvidia/blueprint/charts/nvidia-blueprint-rag-v2.6.0.tgz \
  -n rag \
  --username '$oauthtoken' \
  --password "$SUSE_AIF_NVIDIA_API_KEY" \
  -f mig-slicing/values-mig-h100.yaml
```

When credentials need refreshing:

```bash
--set-string imagePullSecret.password="$SUSE_AIF_NVIDIA_API_KEY"
--set-string ngcApiSecret.password="$SUSE_AIF_NVIDIA_API_KEY"
```

---

# 18. What Not to Delete

During troubleshooting, preserve the following unless a deliberate full rebuild is required:

```text
Helm release: rag
NIM cache PVCs
RAG application PVCs
Elasticsearch PVCs
Redis PVCs
SeaweedFS PVCs
```

For an immutable NIMCache model change, the object may need to be recreated:

```bash
kubectl delete nimcache nim-llm-cache -n rag
```

but preserve:

```text
nim-llm-cache-pvc
```

Do not use `helm upgrade --force` as the first solution for an immutable NIMCache field.

---

# 19. Troubleshooting Decision Tree

## NIMCache is `Pending`

Check:

```bash
kubectl describe nimcache <name> -n rag
```

Then:

```bash
kubectl get events -n rag --sort-by='.lastTimestamp' | tail -100
```

Check NIM Operator logs:

```bash
kubectl logs -n rag deployment/k8s-nim-operator --since=30m
```

Common areas to check:

- NGC authentication.
- Registry connectivity.
- PVC/storage.
- Cache Job creation and scheduling.

## NIMCache shows 401 Unauthorized

Check:

```bash
kubectl get secret ngc-api -n rag \
  -o jsonpath='{.data.NGC_API_KEY}' | base64 -d | wc -c
```

If the value is `0`, refresh the Secret through Helm using the current NGC API key.

## NIMService is `NotReady` or `Failed`

Check:

```bash
kubectl get nimservice <name> -n rag -o yaml
```

Then:

```bash
kubectl get pods -n rag | grep <name>
```

Then inspect:

```bash
kubectl logs -n rag <pod>
kubectl logs -n rag <pod> --previous
```

## GPU resource says 2 but only one GPU is visible

Check:

```bash
kubectl get pod <pod> -n rag \
  -o jsonpath='{.spec.containers[*].resources}'
```

Then:

```bash
kubectl exec -n rag <pod> -- nvidia-smi -L
```

Also:

```bash
kubectl exec -n rag <pod> -- ls -l /dev/nvidia*
```

Do not assume `NVIDIA_VISIBLE_DEVICES=void` is the root cause by itself; validate actual GPU visibility with `nvidia-smi`.

## vLLM fails in PyTorch Inductor

Look for:

```text
torch._inductor.exc.InductorError
AssertionError
expr_fits_within_32bit
```

First verify that the chart's normal NIM environment entries have not been accidentally replaced by a shortened custom `env:` list.

The final working configuration restored:

```text
NIM_HTTP_API_PORT
NIM_TRITON_LOG_VERBOSE
NIM_SERVED_MODEL_NAME
NIM_ENABLE_CHUNKED_PREFILL
NCCL_NVLS_ENABLE
VLLM_USE_FLASHINFER_MOE_FP8
NIM_PASSTHROUGH_ARGS
```

---

# 20. Final Outcome

The successful configuration has:

```text
Helm chart           -> RAG Blueprint v2.6.0
Nemotron NIM         -> 2.0.9
LLM GPUs             -> 2 × full H100 NVL
Tensor Parallelism   -> 2
Reranker             -> 1 × MIG 1g.24gb
OCR                  -> 1 × MIG 3g.47gb
VLM Embedding        -> 1 × MIG 3g.47gb
Graphic              -> 1 × MIG 1g.24gb
Page                 -> 1 × MIG 1g.24gb
Table                -> 1 × MIG 1g.24gb
max-num-seqs         -> 384
Chunked prefill      -> enabled
```

The main lessons from the troubleshooting were:

1. An immutable NIMCache model reference requires recreating the NIMCache object rather than patching the immutable field.
2. Preserve the NIMCache PVC when possible.
3. Offline `helm template` may need `--api-versions apps.nvidia.com/v1alpha1` to render NIM Operator resources.
4. Verify both the NGC API secret and the registry pull secret.
5. Keep the chart's normal NIM environment entries when adding custom runtime arguments.
6. Validate actual GPU visibility inside the container instead of relying only on Kubernetes resource requests/limits.
7. Check the NIMCache first, then NIMService status, then container logs for runtime failures.


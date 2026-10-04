# NVIDIA RAG Blueprint v2.6.0 — H100 NVL MIG Deployment & Helm Troubleshooting

## Environment

Target server:

- 4 × NVIDIA H100 NVL
- ~94 GB HBM per GPU
- NVIDIA driver: 580.178.04
- CUDA: 13.0
- OpenShift/Kubernetes environment
- NVIDIA RAG Blueprint v2.6.0

Recommended high-level GPU layout:

```text
GPU 0  FULL 94 GB ─┐
                    ├── LLM / vLLM / Tensor Parallel = 2
GPU 1  FULL 94 GB ─┘

GPU 2  MIG
       ├── 3g.47gb
       ├── 1g.24gb
       └── 1g.24gb

GPU 3  MIG
       ├── 3g.47gb
       ├── 1g.24gb
       └── 1g.24gb
```

Logical RAG allocation:

| RAG component | GPU resource | Count |
|---|---|---:|
| Nemotron 120B LLM | `nvidia.com/gpu` | 2 |
| OCR | `nvidia.com/mig-3g.47gb` | 1 |
| VLM embedding | `nvidia.com/mig-3g.47gb` | 1 |
| Graphic Elements | `nvidia.com/mig-1g.24gb` | 1 |
| Page Elements | `nvidia.com/mig-1g.24gb` | 1 |
| Table Structure | `nvidia.com/mig-1g.24gb` | 1 |
| Reranker | `nvidia.com/mig-1g.24gb` | 1 |

> Important: NVIDIA's published RAG MIG example is based on H100 80 GB. Your H100 NVL is the 94 GB variant, so do not copy the 80 GB profile names such as `3g.40gb` or `1g.10gb`.

---

# 1. Verify the actual H100 NVL MIG profiles

Before configuring MIG, run:

```bash
nvidia-smi mig -lgip
nvidia-smi mig -lgipp
```

These commands show the GPU-instance profiles and valid placement combinations exposed by the installed driver.

For H100 94 GB, expected profiles include:

```text
1g.12gb
1g.24gb
2g.24gb
3g.47gb
4g.47gb
7g.94gb
```

Do not proceed with a production configuration until the exact profiles and placement combinations are confirmed on the server.

---

# 2. Recommended MIG architecture

Keep GPU 0 and GPU 1 as full GPUs.

They are intended for the LLM:

```text
GPU 0 → Full H100 NVL
GPU 1 → Full H100 NVL
```

The LLM uses:

```text
vLLM
FP8
Tensor Parallelism = 2
```

Configure GPU 2 and GPU 3 with MIG:

```text
GPU 2:
  3g.47gb
  1g.24gb
  1g.24gb

GPU 3:
  3g.47gb
  1g.24gb
  1g.24gb
```

This provides:

```text
2 × full GPUs
2 × 3g.47gb MIG instances
4 × 1g.24gb MIG instances
```

---

# 3. NVIDIA GPU Operator

The GPU Operator should use the `mixed` MIG strategy because the node exposes both full GPUs and MIG resources.

Check the current ClusterPolicy:

```bash
kubectl get clusterpolicy cluster-policy -o yaml
```

The relevant configuration should be:

```yaml
spec:
  mig:
    strategy: mixed
```

If required:

```bash
kubectl patch clusterpolicies.nvidia.com/cluster-policy \
  --type='json' \
  -p='[
    {
      "op": "replace",
      "path": "/spec/mig/strategy",
      "value": "mixed"
    }
  ]'
```

---

# 4. Verify node GPU resources

After MIG configuration is applied, check:

```bash
oc describe node hclaif | grep -E \
'nvidia.com/gpu|nvidia.com/mig'
```

Expected logical resources:

```text
nvidia.com/gpu:             2
nvidia.com/mig-3g.47gb:     2
nvidia.com/mig-1g.24gb:     4
```

Also verify at the NVIDIA level:

```bash
nvidia-smi
nvidia-smi -L
```

---

# 5. NVIDIA RAG Blueprint Helm chart

Target chart:

```text
nvidia-blueprint-rag-v2.6.0
```

Example installation:

```bash
helm upgrade --install rag -n rag \
  https://helm.ngc.nvidia.com/nvidia/blueprint/charts/nvidia-blueprint-rag-v2.6.0.tgz \
  --username '$oauthtoken' \
  --password "${SUSE_AIF_NVIDIA_API_KEY}" \
  --set imagePullSecret.password="$SUSE_AIF_NVIDIA_API_KEY" \
  --set ngcApiSecret.password="$SUSE_AIF_NVIDIA_API_KEY" \
  --set nimOperator.nim-llm.image.tag=2.0.9 \
  -f mig-slicing/values-mig-h100.yaml
```

---

# 6. Helm error encountered with `env[6]`

The following approach should NOT be used:

```bash
--set 'nimOperator.nim-llm.env[6].name=NIM_PASSTHROUGH_ARGS' \
--set-string 'nimOperator.nim-llm.env[6].value=--max-num-seqs 384'
```

Why?

Helm list indexes are positional. If the chart does not already contain entries at indexes 0–5, setting index 6 can create a sparse array:

```yaml
env:
  - null
  - null
  - null
  - null
  - null
  - null
  - name: NIM_PASSTHROUGH_ARGS
    value: "--max-num-seqs 384"
```

The NVIDIA NIM Operator CRD rejects these null objects.

The resulting error looks like:

```text
spec.env[0]: Invalid value: "null"
spec.env[1]: Invalid value: "null"
...
spec.env[5]: Invalid value: "null"
```

---

# 7. Inspect the actual chart values first

Before modifying the environment list, inspect the v2.6.0 chart:

```bash
helm show values \
  https://helm.ngc.nvidia.com/nvidia/blueprint/charts/nvidia-blueprint-rag-v2.6.0.tgz \
  > /tmp/rag-values.yaml
```

Then:

```bash
grep -n -A100 -B10 "nim-llm:" /tmp/rag-values.yaml
```

Also inspect your current override file:

```bash
grep -n -A100 -B10 "nim-llm:" \
  mig-slicing/values-mig-h100.yaml
```

Look specifically for:

```yaml
nimOperator:
  nim-llm:
    env:
```

---

# 8. Safely add `NIM_PASSTHROUGH_ARGS`

If the values file already defines an `env` list, append the environment variable to that existing list.

For example:

```yaml
nimOperator:
  nim-llm:
    env:
      # Existing environment entries must remain here
      # ...

      - name: NIM_PASSTHROUGH_ARGS
        value: "--max-num-seqs 384"
```

Do not create a sparse Helm array with:

```bash
env[6]
```

unless you have confirmed that entries 0–5 already exist.

---

# 9. Validate Helm rendering before installation

Render the chart first:

```bash
helm template rag \
  https://helm.ngc.nvidia.com/nvidia/blueprint/charts/nvidia-blueprint-rag-v2.6.0.tgz \
  -n rag \
  -f mig-slicing/values-mig-h100.yaml \
  > rendered-rag.yaml
```

Then inspect the generated NIMService:

```bash
grep -n -A30 -B10 "NIM_PASSTHROUGH_ARGS" rendered-rag.yaml
```

Check for null environment entries:

```bash
grep -n "env:" rendered-rag.yaml
grep -n "null" rendered-rag.yaml
```

Also inspect GPU resource requests:

```bash
grep -n -A10 -B5 "mig-" rendered-rag.yaml
```

---

# 10. Recommended RAG resource mapping

The intended application-level mapping is:

```text
FULL GPU
────────
GPU 0 → LLM / vLLM TP rank 0
GPU 1 → LLM / vLLM TP rank 1


3g.47gb MIG
───────────
1 → OCR
1 → VLM embedding


1g.24gb MIG
───────────
1 → Graphic Elements
1 → Page Elements
1 → Table Structure
1 → Reranker
```

Kubernetes sees the resources logically:

```text
nvidia.com/gpu              2
nvidia.com/mig-3g.47gb      2
nvidia.com/mig-1g.24gb      4
```

The Kubernetes scheduler does not necessarily bind a particular `3g.47gb` resource to a specific physical GPU unless additional topology/affinity controls are used.

---

# 11. Important performance consideration

The upstream RAG Blueprint reserves a full H100 for the VLM embedding workload in its larger GPU configuration.

With the four-GPU H100 NVL configuration above, the VLM embedding workload is placed on:

```text
nvidia.com/mig-3g.47gb
```

This is appropriate for a POC or development environment, but should be benchmarked for production ingestion throughput.

If VLM throughput becomes the bottleneck, an alternative configuration is:

```text
GPU 0 → FULL → LLM
GPU 1 → FULL → LLM
GPU 2 → FULL → VLM embedding
GPU 3 → MIG  → OCR / reranker / other services
```

This more closely resembles NVIDIA's larger-GPU RAG architecture, but sacrifices MIG capacity.

---

# 12. Suggested troubleshooting sequence

If Helm fails:

### Step 1 — Inspect chart

```bash
helm show values \
  https://helm.ngc.nvidia.com/nvidia/blueprint/charts/nvidia-blueprint-rag-v2.6.0.tgz \
  > /tmp/rag-values.yaml
```

### Step 2 — Inspect LLM configuration

```bash
grep -n -A100 -B10 "nim-llm:" /tmp/rag-values.yaml
```

### Step 3 — Inspect custom values

```bash
grep -n -A100 -B10 "nim-llm:" \
  mig-slicing/values-mig-h100.yaml
```

### Step 4 — Render

```bash
helm template rag \
  https://helm.ngc.nvidia.com/nvidia/blueprint/charts/nvidia-blueprint-rag-v2.6.0.tgz \
  -n rag \
  -f mig-slicing/values-mig-h100.yaml \
  > rendered-rag.yaml
```

### Step 5 — Inspect NIM environment

```bash
grep -n -A30 -B10 "NIM_PASSTHROUGH_ARGS" rendered-rag.yaml
```

### Step 6 — Inspect GPU resources

```bash
grep -n -A10 -B5 "mig-" rendered-rag.yaml
```

### Step 7 — Check NVIDIA resources

```bash
nvidia-smi
nvidia-smi -L
```

### Step 8 — Check Kubernetes resources

```bash
oc describe node hclaif | grep -E \
'nvidia.com/gpu|nvidia.com/mig'
```

---

# 13. Current error diagnosis

Your original command included:

```bash
--set 'nimOperator.nim-llm.env[6].name=NIM_PASSTHROUGH_ARGS'
--set-string 'nimOperator.nim-llm.env[6].value=--max-num-seqs 384'
```

The error:

```text
NIMService.apps.nvidia.com "nim-llm" is invalid:
spec.env[0]: Invalid value: "null"
...
spec.env[5]: Invalid value: "null"
```

is caused by the sparse environment array generated by those Helm overrides.

Remove those two options from the command and put the environment variable into the values file after inspecting the chart's existing environment list.

---

# 14. Immediate next commands

Run:

```bash
helm show values \
  https://helm.ngc.nvidia.com/nvidia/blueprint/charts/nvidia-blueprint-rag-v2.6.0.tgz \
  > /tmp/rag-values.yaml
```

Then:

```bash
grep -n -A100 -B10 "nim-llm:" /tmp/rag-values.yaml
```

And:

```bash
grep -n -A100 -B10 "nim-llm:" \
  mig-slicing/values-mig-h100.yaml
```

Also verify your H100 NVL MIG profiles:

```bash
nvidia-smi mig -lgip
nvidia-smi mig -lgipp
```

These outputs should be used to finalize the exact `values-mig-h100.yaml` and MIG configuration rather than copying the H100 80 GB NVIDIA example unchanged.

---

## Final target architecture

```text
                         H100 NVL × 4
                              │
          ┌───────────────────┴───────────────────┐
          │                                       │
      FULL GPUs                              MIG GPUs
          │                                       │
    ┌─────┴─────┐                    ┌────────────┴────────────┐
    │           │                    │                         │
  GPU 0       GPU 1                GPU 2                     GPU 3
  94 GB       94 GB                 MIG                       MIG
    │           │                    │                         │
    └─────┬─────┘              3g.47 + 1g + 1g          3g.47 + 1g + 1g
          │
       LLM TP=2

RAG services:

3g.47gb → OCR
3g.47gb → VLM Embedding

1g.24gb → Graphic Elements
1g.24gb → Page Elements
1g.24gb → Table Structure
1g.24gb → Reranker
```

This is the baseline configuration to benchmark for your 4 × H100 NVL RAG Blueprint deployment.


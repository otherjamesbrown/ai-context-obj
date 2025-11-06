# Spec 010: vLLM Deployment

## Purpose

Deploy vLLM inference engines on GPU node pool and register models in the platform.

## Dependencies

- **001-infrastructure**: GPU node pool available
- **009-admin-cli**: For registering models

## Success Criteria

- [ ] vLLM Helm chart configured
- [ ] At least one model (Llama 3 8B) deployed
- [ ] Model accessible at K8s service URL
- [ ] Model registered in model_mappings table
- [ ] Health check endpoint working
- [ ] GPU utilization monitored

## Models for V1

### Llama 3 8B (Primary)
- HuggingFace: meta-llama/Meta-Llama-3-8B-Instruct
- GPU Memory: ~16GB
- Max Tokens: 8192

## Helm Configuration

### vLLM Helm Chart
Use community chart or create custom:

```yaml
# deployments/charts/vllm-llama3/values.yaml
image:
  repository: vllm/vllm-openai
  tag: latest

model:
  name: meta-llama/Meta-Llama-3-8B-Instruct
  huggingface_token: ${HF_TOKEN}  # For gated models

resources:
  limits:
    nvidia.com/gpu: 1
    memory: 24Gi
  requests:
    nvidia.com/gpu: 1
    memory: 20Gi

nodeSelector:
  gpu-type: rtx-6000-ada

service:
  name: llama-3-8b-inference
  port: 8000
  type: ClusterIP

env:
  - name: VLLM_TENSOR_PARALLEL_SIZE
    value: "1"
  - name: VLLM_GPU_MEMORY_UTILIZATION
    value: "0.95"

livenessProbe:
  httpGet:
    path: /health
    port: 8000
  initialDelaySeconds: 300
  periodSeconds: 30

readinessProbe:
  httpGet:
    path: /health
    port: 8000
  initialDelaySeconds: 120
  periodSeconds: 10
```

## Deployment Steps

### 1. Deploy vLLM via Helm
```bash
helm install llama3-8b ./deployments/charts/vllm-llama3 \
  --namespace ias-production \
  --set model.huggingface_token=${HF_TOKEN}
```

### 2. Verify Deployment
```bash
kubectl get pods -n ias-production | grep llama3
kubectl logs -n ias-production <pod-name>
```

### 3. Test Inference Endpoint
```bash
kubectl port-forward -n ias-production svc/llama-3-8b-inference 8000:8000

curl http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "meta-llama/Meta-Llama-3-8B-Instruct",
    "messages": [{"role": "user", "content": "Hello!"}]
  }'
```

### 4. Register Model in Platform
```bash
ias-admin model add \
  --name llama3 \
  --display-name "Llama 3 8B Instruct" \
  --backend-url http://llama-3-8b-inference.ias-production.svc.cluster.local:8000 \
  --input-cost 0.0005 \
  --output-cost 0.0015
```

## Model Registry Process

For each new model:
1. Deploy vLLM with Helm (GPU node pool)
2. Create K8s Service (ClusterIP)
3. Register in model_mappings table via CLI
4. Test via API Router

## Adding Additional Models

### Mistral 7B (Example)
```bash
# Deploy
helm install mistral-7b ./deployments/charts/vllm-mistral \
  --namespace ias-production

# Register
ias-admin model add \
  --name mistral-7b \
  --display-name "Mistral 7B Instruct" \
  --backend-url http://mistral-7b-inference.ias-production.svc.cluster.local:8000 \
  --input-cost 0.0003 \
  --output-cost 0.0009
```

## Monitoring

### GPU Metrics
- GPU utilization (%)
- GPU memory usage (GB)
- Requests per second
- Avg latency per request

### vLLM Metrics
vLLM exposes Prometheus metrics at `/metrics`:
- `vllm_requests_total`
- `vllm_request_duration_seconds`
- `vllm_num_requests_running`
- `vllm_gpu_cache_usage_perc`

## Autoscaling (V2)

For future consideration:
- Horizontal Pod Autoscaler based on GPU utilization
- Model-specific scaling policies
- Cold start mitigation

## Contracts

See `contracts/` for:
- `helm-values/` - Helm chart values for each model
- `model-registry.md` - How to add new models
- `gpu-requirements.md` - GPU sizing guide

## Security

- vLLM services only accessible within cluster (ClusterIP)
- Network policies enforce API Router is only ingress
- HuggingFace tokens stored as K8s Secrets

## Next Steps

After vLLM deployed:
- Test end-to-end inference via API Router
- Run E2E tests (spec 012)


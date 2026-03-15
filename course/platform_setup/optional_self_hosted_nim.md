# Self-Hosted NIM (Optional / Advanced)

---

> **THIS GUIDE IS ENTIRELY OPTIONAL.**
>
> Self-hosted NIM is NOT required for any lab in this course. Every lab works with NVIDIA hosted APIs and an `NVIDIA_API_KEY`. This guide exists for learners who have access to GPU hardware and want to understand the self-hosting path, or for reference when studying production deployment architectures (e.g., Module 10).
>
> **Do not attempt self-hosted deployment unless you have the hardware listed below.**

---

## Hardware / Environment Requirements

| Scenario | Hardware | Use Case |
|---|---|---|
| **Small models (7B-13B)** | 1x A100 40GB or 1x H100 80GB | Development, testing, experimentation |
| **Medium models (34B-40B)** | 1x A100 80GB or 1x H100 80GB | Development, moderate production |
| **Large models (70B)** | 2x H100 80GB (NVLink) or 2x A100 80GB | Production inference |
| **Multi-model serving** | 4+ GPUs or Kubernetes cluster | Production: LLM + embedding + reranker simultaneously |
| **NOT required for this course** | CPU laptop + NVIDIA_API_KEY | All labs, all modules |

**Software requirements for self-hosted NIM:**
- Linux OS (Ubuntu 22.04 recommended)
- NVIDIA GPU driver 535+ (check with `nvidia-smi`)
- Docker 24+
- NVIDIA Container Toolkit (`nvidia-container-toolkit`)
- NGC account and API key (separate from the build.nvidia.com API key)

---

## Prerequisites

- Completed: [NVIDIA Account and API Key Setup](build_nvidia_account_and_api_key.md)
- Completed: [First NIM API Call](first_nim_api_call.md) (to understand the API you are self-hosting)
- A Linux machine with NVIDIA GPU(s) meeting the specs above
- Docker installed and running
- NVIDIA Container Toolkit installed and configured

---

## Setup Steps

### Step 1: What Self-Hosted NIM Is

NIM (NVIDIA Inference Microservice) is a containerized, GPU-optimized inference runtime. When you use the hosted API at `integrate.api.nvidia.com`, NVIDIA runs NIM containers on their infrastructure. Self-hosted NIM means you run those same containers on your own GPU hardware.

**The key insight**: The API is identical. Code that works against `integrate.api.nvidia.com/v1` works against `localhost:8000/v1` with zero changes (just swap the base URL). This is by design — NIM provides a consistent API whether hosted or self-hosted.

What NIM includes:
- Optimized model weights (quantized, compiled for specific GPU architectures)
- TensorRT-LLM or vLLM backend for high-throughput inference
- OpenAI-compatible REST API
- Health checks, metrics endpoints, and logging
- Automatic GPU memory management and batching

### Step 2: When You Need Self-Hosted NIM

| Requirement | Hosted API | Self-Hosted NIM |
|---|---|---|
| Getting started / prototyping | Best choice | Overkill |
| Low latency (< 100ms TTFT) | Network adds latency | Achievable |
| Data privacy / compliance | Data goes to NVIDIA cloud | Data stays on-premise |
| Offline / air-gapped operation | Not possible | Supported |
| Cost at scale (millions of tokens/day) | Per-token pricing | Fixed infrastructure cost |
| Custom model (fine-tuned) | Not supported (unless uploaded) | Full control |
| Guaranteed availability / SLA | Depends on NVIDIA's infrastructure | You own the SLA |

**Rule of thumb**: Use hosted APIs for learning, prototyping, and low-volume production. Use self-hosted NIM when you have latency, privacy, cost-at-scale, or custom model requirements.

### Step 3: Hardware Requirements by Model Size

Reference table based on NIM documentation and AI-Q Blueprint specifications:

| Model Size | Minimum GPU | Recommended GPU | Approximate GPU Memory |
|---|---|---|---|
| 7B parameters | 1x A10G 24GB | 1x A100 40GB | ~14GB (FP16), ~7GB (INT8) |
| 13B parameters | 1x A100 40GB | 1x A100 80GB | ~26GB (FP16), ~13GB (INT8) |
| 34B parameters | 1x A100 80GB | 1x H100 80GB | ~68GB (FP16), ~34GB (INT8) |
| 70B parameters | 2x A100 80GB | 2x H100 80GB | ~140GB (FP16), ~70GB (INT8) |
| Embedding models | 1x T4 16GB | 1x A10G 24GB | ~2-4GB |
| Reranker models | 1x T4 16GB | 1x A10G 24GB | ~2-4GB |

NIM uses TensorRT-LLM optimizations (quantization, KV cache management, continuous batching) to maximize throughput within the available GPU memory.

### Step 4: NGC (NVIDIA GPU Cloud) Access

NGC is NVIDIA's container registry and model hub. Self-hosted NIM containers are distributed via NGC.

**Create an NGC account:**

1. Go to `https://ngc.nvidia.com`
2. Sign up or sign in (you can use the same NVIDIA account as build.nvidia.com)
3. Navigate to your user profile (top-right menu)
4. Click **Setup** > **Generate API Key**
5. Copy the NGC API key. This is different from your `NVIDIA_API_KEY` for hosted APIs.

**Store the NGC API key:**

```bash
export NGC_API_KEY="your-ngc-api-key-here"
```

**Log in to the NGC container registry:**

```bash
echo "$NGC_API_KEY" | docker login nvcr.io --username '$oauthtoken' --password-stdin
```

Expected output:

```
Login Succeeded
```

Note: The username is literally `$oauthtoken` (a dollar sign followed by the word oauthtoken). This is not a variable — it is the required username for NGC registry authentication.

### Step 5: Pull a NIM Container

```bash
# Example: Pull the Llama 3.1 8B NIM container (smaller model for testing)
docker pull nvcr.io/nim/meta/llama-3.1-8b-instruct:latest
```

This download can be large (10-30 GB depending on the model). Allow time and ensure you have sufficient disk space.

Verify the image was pulled:

```bash
docker images | grep nim
```

### Step 6: Run the NIM Container

```bash
docker run -d \
  --name nim-llm \
  --gpus all \
  -p 8000:8000 \
  -e NVIDIA_API_KEY="$NGC_API_KEY" \
  nvcr.io/nim/meta/llama-3.1-8b-instruct:latest
```

Flags explained:
- `--gpus all`: Pass all available GPUs to the container (requires NVIDIA Container Toolkit)
- `-p 8000:8000`: Map container port 8000 to host port 8000
- `-e NVIDIA_API_KEY`: Authentication for model weight downloads (some NIM containers download optimized weights on first start)

**Wait for the container to become ready** (this can take 2-10 minutes on first run as the model loads into GPU memory):

```bash
# Check container logs
docker logs -f nim-llm
```

Look for a line like:

```
INFO: Application startup complete.
INFO: Uvicorn running on http://0.0.0.0:8000
```

**Health check:**

```bash
curl -s http://localhost:8000/v1/health/ready
```

Expected response:

```json
{"status": "ready"}
```

### Step 7: Test the Local NIM Endpoint

The API is identical to the hosted endpoint. Only the URL changes.

```bash
curl -s http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "meta/llama-3.1-8b-instruct",
    "messages": [
      {"role": "user", "content": "What is NVIDIA NIM in one sentence?"}
    ],
    "max_tokens": 100,
    "temperature": 0.7
  }'
```

**Using the OpenAI Python client (identical code, different base_url):**

```python
from openai import OpenAI

# For hosted API:
# client = OpenAI(base_url="https://integrate.api.nvidia.com/v1", api_key=os.environ["NVIDIA_API_KEY"])

# For self-hosted NIM:
client = OpenAI(base_url="http://localhost:8000/v1", api_key="not-needed-for-local")

response = client.chat.completions.create(
    model="meta/llama-3.1-8b-instruct",
    messages=[{"role": "user", "content": "What is NVIDIA NIM in one sentence?"}],
    max_tokens=100,
    temperature=0.7,
)

print(response.choices[0].message.content)
```

This demonstrates the core NIM value proposition: **write once, deploy anywhere**. Your application code does not change between hosted and self-hosted.

### Step 8: NIM Operator for Kubernetes

For production deployments with multiple models, replicas, and autoscaling, NVIDIA provides the **NIM Operator** for Kubernetes.

**What it does:**
- Manages NIM containers as Kubernetes custom resources
- Handles GPU scheduling and allocation
- Supports autoscaling based on request load
- Manages model versioning and rolling updates
- Provides observability (metrics, logs, tracing)

**When to use it:**
- Multi-model serving (LLM + embedding + reranker on the same cluster)
- High availability requirements (multiple replicas with load balancing)
- Dynamic scaling (scale down during low traffic, scale up during peaks)
- Production SLA enforcement

**Basic concept (you do not need to deploy this now):**

```yaml
# Example NIM Operator custom resource (conceptual)
apiVersion: nim.nvidia.com/v1
kind: NIMService
metadata:
  name: llama-3-1-8b
spec:
  model: meta/llama-3.1-8b-instruct
  replicas: 2
  resources:
    gpus: 1
    gpuType: A100
  autoscaling:
    minReplicas: 1
    maxReplicas: 4
    targetUtilization: 70
```

Full Kubernetes deployment is covered in Module 10 of this course. This step is here for architectural awareness only.

### Step 9: NeMo Microservices Deployment Concepts

Beyond NIM (inference), NVIDIA provides additional microservices for the full AI lifecycle:

| Microservice | Function | When You Need It |
|---|---|---|
| **NIM** | Optimized inference | Always (for self-hosted inference) |
| **NeMo Customizer** | Fine-tuning models (LoRA, P-tuning) | When you need to adapt a model to your domain |
| **NeMo Evaluator** | Automated model evaluation | When comparing model versions or benchmarking |
| **NeMo Retriever** | Document processing, embedding, retrieval | When building RAG pipelines with enterprise documents |
| **NeMo Guardrails** | Safety and content moderation | When deploying to production with safety requirements. Note: NeMo Guardrails also works in **library mode** (`pip install nemoguardrails langchain-nvidia-ai-endpoints`) on a CPU laptop — container deployment is only needed for microservice mode in production. |

These are all containerized microservices that can be deployed alongside NIM. In the AI-Q Blueprint (see the Blueprint Walkthrough), several of these run together via Docker Compose.

Key deployment patterns:
- **Docker Compose**: For single-node deployments with a few services. Good for development and small-scale production.
- **Kubernetes with NIM Operator**: For multi-node, autoscaled, production-grade deployments.
- **Hybrid**: NIM self-hosted for inference (latency, privacy) + hosted APIs for customization and evaluation.

### Step 10: Cost and Complexity Analysis

| Factor | Hosted API | Self-Hosted NIM |
|---|---|---|
| **Upfront cost** | $0 | $10,000-$200,000+ (GPU hardware) or cloud GPU rental ($2-$30/hr) |
| **Per-token cost** | Pay per token | $0 marginal (amortized hardware) |
| **Break-even point** | Below ~1M tokens/day | Above ~1M tokens/day (depends on hardware cost) |
| **Setup time** | Minutes | Hours to days |
| **Maintenance** | None (NVIDIA manages) | You manage: updates, monitoring, GPU health, scaling |
| **Expertise needed** | API integration | Docker, GPUs, networking, monitoring, Kubernetes (for production) |
| **Latency** | 200-500ms TTFT (network dependent) | 50-150ms TTFT (local) |
| **Data privacy** | Data sent to NVIDIA | Data stays on your infrastructure |
| **Offline operation** | Not possible | Fully supported |

**Decision framework:**

1. **Start with hosted APIs.** Always. For learning, prototyping, and low-volume production.
2. **Evaluate self-hosting when** you hit one of: latency requirements below 200ms, data privacy/compliance mandates, cost exceeding $X,000/month on hosted APIs, or need for offline operation.
3. **Start self-hosting small.** Deploy one model on one GPU. Measure actual performance before scaling.
4. **Move to Kubernetes** only when you need multi-model serving, autoscaling, or high availability.

---

## Verification Checklist

If you completed the optional self-hosted deployment:

- [ ] `nvidia-smi` shows your GPU(s) with driver version 535+
- [ ] `docker --version` shows 24+
- [ ] `docker login nvcr.io` succeeds
- [ ] NIM container pulled successfully
- [ ] Container starts and `health/ready` endpoint returns `{"status": "ready"}`
- [ ] curl to `localhost:8000/v1/chat/completions` returns a model response
- [ ] OpenAI Python client works against `localhost:8000/v1`
- [ ] GPU utilization visible in `nvidia-smi` during inference

If you did NOT deploy (which is fine):

- [ ] You understand what self-hosted NIM is and when it is appropriate
- [ ] You can explain the difference between hosted API and self-hosted NIM
- [ ] You understand the hardware requirements for different model sizes
- [ ] You know what NGC is and how container registry authentication works
- [ ] You understand the NIM Operator's role in Kubernetes deployments
- [ ] You can articulate the cost trade-offs between hosted and self-hosted

### Expected Successful Output (Step 7 curl)

```json
{
  "id": "chatcmpl-xxxxxxxx",
  "object": "chat.completion",
  "created": 1700000000,
  "model": "meta/llama-3.1-8b-instruct",
  "choices": [
    {
      "index": 0,
      "message": {
        "role": "assistant",
        "content": "NVIDIA NIM is a set of GPU-accelerated inference microservices that enable developers to deploy optimized AI models as scalable, containerized API endpoints."
      },
      "finish_reason": "stop"
    }
  ],
  "usage": {
    "prompt_tokens": 15,
    "completion_tokens": 32,
    "total_tokens": 47
  }
}
```

---

## Common Failure Cases

### 1. Insufficient GPU Memory

**Symptom**: Container crashes on startup with `CUDA out of memory` or `RuntimeError: CUDA error: out of memory`.

**Fix**:
- Check your GPU memory: `nvidia-smi` (look at "Memory-Usage" column).
- Use a smaller model (7B or 8B) if you have less than 40GB GPU memory.
- Close other GPU-consuming processes.
- NIM uses quantization by default, but the model still needs to fit. A 7B model needs approximately 7-14GB depending on precision.

### 2. Docker Not Configured for GPU Access

**Symptom**: Container starts but `nvidia-smi` inside the container fails, or the container immediately exits with GPU-related errors.

**Fix**:
- Install NVIDIA Container Toolkit:
  ```bash
  # Ubuntu/Debian
  sudo apt-get install -y nvidia-container-toolkit
  sudo systemctl restart docker
  ```
- Verify GPU access in Docker:
  ```bash
  docker run --rm --gpus all nvidia/cuda:12.0-base nvidia-smi
  ```
  This should print your GPU information. If it fails, the toolkit is not configured correctly.
- Check the Docker daemon configuration: `/etc/docker/daemon.json` should include the nvidia runtime.

### 3. NGC Authentication Issues

**Symptom**: `docker pull` fails with `unauthorized` or `authentication required`.

**Fix**:
- Confirm you logged in: `docker login nvcr.io`
- The username must be exactly `$oauthtoken` (literal dollar sign and word).
- The password is your NGC API key (not your NVIDIA account password, not your build.nvidia.com API key).
- Generate a new NGC API key at `ngc.nvidia.com` if the current one is expired.

### 4. Port Conflicts

**Symptom**: `Bind for 0.0.0.0:8000 failed: port is already allocated`.

**Fix**:
- Check what is using port 8000: `lsof -i :8000` or `ss -tlnp | grep 8000`
- Use a different port: `-p 8001:8000` in the docker run command, then access at `localhost:8001`.
- Kill the conflicting process if it is an old NIM container: `docker stop <container_name>`.

### 5. Model Download Timeouts

**Symptom**: Container hangs during startup with messages about downloading model weights. Eventually times out or fails.

**Fix**:
- First-run downloads can be 10-50 GB. Ensure you have a fast, stable internet connection.
- Use a Docker volume to persist downloaded models across container restarts:
  ```bash
  docker run -d \
    --name nim-llm \
    --gpus all \
    -p 8000:8000 \
    -v nim-cache:/opt/nim/cache \
    -e NVIDIA_API_KEY="$NGC_API_KEY" \
    nvcr.io/nim/meta/llama-3.1-8b-instruct:latest
  ```
  The `-v nim-cache:/opt/nim/cache` flag persists the model weights so subsequent starts are fast.
- If behind a corporate proxy, configure Docker's proxy settings in `/etc/docker/daemon.json` or `~/.docker/config.json`.

### 6. NVIDIA Driver Version Too Old

**Symptom**: Container starts but CUDA initialization fails with version mismatch errors.

**Fix**:
- Check your driver version: `nvidia-smi` (top of output shows "Driver Version").
- NIM containers typically require driver 535 or newer.
- Update your driver: follow NVIDIA's official driver installation guide for your OS.
- After updating, reboot and verify: `nvidia-smi`.

### 7. Container Runs but API Returns Errors

**Symptom**: Health check passes but `/v1/chat/completions` returns 500 or model-specific errors.

**Fix**:
- Check container logs: `docker logs nim-llm`
- Look for model loading errors (quantization failures, incompatible model format).
- Ensure you are using the correct model name in your API request (it must match the container's model).
- Try reducing `max_tokens` to rule out memory pressure during generation.

---

## Cleanup / Cost-Awareness Notes

- **Stop containers when not in use**: GPU memory is consumed as long as the container runs.
  ```bash
  docker stop nim-llm
  docker rm nim-llm    # Remove the container (model cache in volume persists)
  ```
- **Cloud GPU costs**: If you rented a cloud GPU instance (AWS p4d, GCP a2, Azure ND, etc.), shut it down when done. GPU instances cost $2-$30+/hour depending on GPU type.
  ```bash
  # Verify no containers are running
  docker ps
  # Then shut down/terminate your cloud instance via your provider's console
  ```
- **Disk space**: NIM container images and model caches can consume 20-100 GB. Clean up with:
  ```bash
  docker system prune -a    # Removes all unused containers and images
  docker volume prune        # Removes unused volumes (including model cache)
  ```
- **NGC API key security**: Treat your NGC API key like a password. It grants access to pull container images. Revoke it at `ngc.nvidia.com` if compromised.
- **Course labs do NOT require self-hosted NIM**: If you deployed NIM for this walkthrough, you can safely shut everything down. All subsequent labs use hosted APIs.

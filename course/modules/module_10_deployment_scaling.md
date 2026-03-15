# Module 10: Deployment and Scaling

## A. Module Title

**Deployment and Scaling of Agentic AI Systems**

Primary Exam Domain: Deployment and Scaling (13%)

---

## B. Why This Module Matters in Real Systems

An agent that runs in a Jupyter notebook is a prototype. An agent that serves 500 concurrent users with sub-second latency, recovers from failures, and costs a predictable amount per request is a product. The gap between these two states is where most agentic AI projects fail. Deployment and scaling is not an afterthought — it is an architectural discipline that must be planned from the first design conversation. The exam allocates 13% of its weight here because NVIDIA treats production-readiness as a core competency, not a DevOps concern.

The deployment challenge for agent systems is structurally different from deploying a standard ML model. A classification model receives a fixed-size input, runs a single forward pass, and returns a fixed-size output. An agent receives a request, plans a multi-step execution, calls tools (each of which may invoke external services or additional LLM calls), manages state across steps, and produces a variable-length response after a variable amount of time. This means every assumption baked into traditional autoscaling (fixed latency, stateless requests, predictable resource consumption) breaks down. You must understand these differences to deploy agents correctly.

NVIDIA provides a specific deployment stack: NAT (NVIDIA Agent Toolkit) deployment servers handle orchestration and API exposure, NIM microservices handle optimized model inference, NIM Operator automates Kubernetes deployment, and NVIDIA Dynamo accelerates inference throughput. Understanding how these components compose — and when to use which deployment server — is directly testable on the exam.

---

## C. Learning Objectives

1. Explain why agent systems require different deployment strategies than stateless APIs
2. Select the correct NAT deployment server (FastAPI, MCP, FastMCP, A2A, Console) for a given production scenario
3. Configure NAT async job management with Dask for background task execution
4. Deploy NIM microservices for LLM inference and connect them to a NAT orchestration layer
5. Design a Kubernetes deployment using NIM Operator and Helm charts with appropriate GPU resource requests
6. Estimate GPU requirements for an agent workload given model sizes, concurrency targets, and latency SLAs
7. Implement health checks, readiness probes, and backpressure mechanisms for agent endpoints
8. Apply autoscaling strategies that account for variable-length agent execution

---

## D. Required Concepts

- Docker and container fundamentals (images, Compose, volumes, networking)
- Kubernetes basics (pods, deployments, services, resource requests/limits)
- REST API design and WebSocket protocols
- GPU memory concepts (VRAM allocation, model loading)
- NAT agent architecture from Modules 3-5
- NeMo Guardrails configuration from Module 8

---

## E. Core Lesson Content

### [BEGINNER] The Deployment Gap

Agent systems that work in notebooks fail in production for five specific reasons:

1. **Latency**: A notebook tolerates 30-second agent runs. A user-facing API does not.
2. **Concurrency**: A notebook runs one request at a time. Production handles hundreds.
3. **Reliability**: A notebook crashes and you restart it. Production must self-heal.
4. **Cost**: A notebook runs on one GPU you already paid for. Production costs scale with traffic.
5. **Security**: A notebook has your API keys in environment variables. Production needs secrets management, authentication, and input validation.

Each of these forces specific architectural decisions. The rest of this module addresses them systematically.

### [BEGINNER] NAT Deployment Servers

NAT provides five deployment frontends. Each wraps the same agent logic but exposes it through a different protocol.

**FastAPI Server** — The standard production deployment. Provides:
- REST API endpoints for synchronous and asynchronous agent invocation
- WebSocket support for streaming token-by-token responses
- Async job management: submit a request, get a job ID, poll for completion
- Health check endpoints (`/health`, `/ready`)
- Static asset serving for any web UI
- OpenAPI documentation auto-generated from agent schema

When to use: Any production deployment where clients call the agent via HTTP. This is the default choice.

**MCP Server** — Implements the Model Context Protocol, making your agent accessible as a tool to other MCP-compatible agents or orchestrators.

When to use: When your agent must be discoverable and callable by other agents in an MCP ecosystem. For example, a specialized "database query agent" that other agents call as a tool.

**FastMCP Server** — A lightweight MCP implementation with less overhead.

When to use: Simpler MCP scenarios where full MCP server features are unnecessary. Development and testing of MCP integrations.

**A2A Server** — Agent-to-Agent protocol server for distributed multi-agent execution. Each agent runs as an independent service, and agents communicate via the A2A protocol.

When to use: Multi-agent systems where agents are deployed as separate microservices, potentially by different teams or on different infrastructure. Enables independent scaling and deployment of each agent.

**Console Frontend** — Terminal-based interactive interface.

When to use: Development and debugging only. Not for production.

```
NAT Deployment Server Selection Decision Tree:
----------------------------------------------
Is this for development/debugging?
  YES → Console Frontend
  NO  → Is the agent consumed by other agents?
          YES → Do agents run as separate services?
                  YES → A2A Server
                  NO  → Is full MCP needed?
                          YES → MCP Server
                          NO  → FastMCP Server
          NO  → FastAPI Server (default production choice)
```

### [INTERMEDIATE] NAT Async Job Management

Long-running agent tasks should not block HTTP connections. NAT integrates with Dask for task queuing and background execution.

The pattern:
1. Client submits a request via POST
2. Server returns a job ID immediately (HTTP 202 Accepted)
3. Dask scheduler queues the agent execution
4. Client polls `/jobs/{job_id}/status` or subscribes via WebSocket
5. When complete, client retrieves the result from `/jobs/{job_id}/result`

This matters because agent executions can take seconds to minutes. Without async job management, you hit HTTP timeout limits, clients retry and create duplicate work, and load balancers kill connections.

Key configuration:
- Dask scheduler type (local threads, distributed)
- Job TTL (how long to keep completed job results)
- Execution history retention for debugging

### [INTERMEDIATE] NIM Microservices for Model Serving

NIM (NVIDIA Inference Microservices) provides containerized, GPU-optimized inference for LLMs and other models. The critical architectural insight: **NAT handles orchestration; NIM handles inference. They are separate concerns.**

```
Architecture: NAT + NIM Separation
====================================

  Client Request
       |
       v
  +-----------------+
  | NAT FastAPI     |   <-- Orchestration layer (CPU)
  | Server          |       Agent logic, tool routing,
  |                 |       state management, guardrails
  +--------+--------+
           |
           | HTTP/gRPC calls for LLM inference
           v
  +-----------------+
  | NIM Container   |   <-- Inference layer (GPU)
  | (e.g., Llama    |       Optimized model serving,
  |  3.1 70B)       |       batching, quantization
  +--------+--------+
           |
           v
     GPU Cluster
  (H100/A100 GPUs)
```

What NIM provides:
- Pre-optimized inference engines (TensorRT-LLM under the hood)
- Automatic batching of concurrent requests
- Quantization options (FP16, INT8, FP8) for memory/speed tradeoffs
- OpenAI-compatible API (drop-in replacement for `openai.ChatCompletion`)
- Health endpoints and metrics export

NIM deployment configuration:
```yaml
# Example: Docker Compose NIM deployment
services:
  nim-llm:
    image: nvcr.io/nim/meta/llama-3.1-70b-instruct:latest
    ports:
      - "8000:8000"
    environment:
      - NGC_API_KEY=${NGC_API_KEY}
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 2
              capabilities: [gpu]
    volumes:
      - nim-cache:/opt/nim/.cache
```

### [INTERMEDIATE] NIM Operator for Kubernetes

NIM Operator automates NIM deployment on Kubernetes clusters. It handles:
- GPU scheduling and resource allocation
- Model cache management across nodes
- Rolling updates without inference downtime
- Autoscaling based on inference queue depth

When to use NIM Operator vs. manual deployment:
- **NIM Operator**: Production K8s clusters, multi-model deployments, teams that want declarative GPU management
- **Manual (Docker/Compose)**: Development, single-node testing, environments without Kubernetes

```yaml
# NIM Operator CRD example
apiVersion: apps.nvidia.com/v1alpha1
kind: NIMService
metadata:
  name: llama-70b-service
spec:
  image: nvcr.io/nim/meta/llama-3.1-70b-instruct:latest
  replicas: 2
  resources:
    limits:
      nvidia.com/gpu: 2
  storage:
    cacheVolume:
      storageClassName: fast-nvme
      size: 200Gi
```

### [INTERMEDIATE] NVIDIA Dynamo

NVIDIA Dynamo is an inference acceleration framework. For agent systems, Dynamo optimizes the inference path that NIM containers use — faster token generation means lower agent latency.

**Note**: Dynamo's agent-specific usage documentation is thinner than its general-purpose docs. The exam is more likely to test your understanding of what Dynamo does conceptually (accelerates inference throughput) than specific Dynamo configuration for agents. Know that Dynamo sits beneath NIM as an optimization layer and that it benefits agent workloads by reducing per-step LLM latency, which compounds across multi-step agent executions.

### [INTERMEDIATE] Containerization Patterns

**Docker Compose for development** — The AI-Q Blueprint documents a reference Compose setup. Minimum hardware: 2x H100 80GB (or 3x A100 80GB). Always verify hardware requirements on the specific Blueprint page or repository before attempting full deployment. Requirements vary significantly across Blueprints. This accommodates:
- NIM containers for LLM inference
- NAT server container for orchestration
- Vector database container (Milvus/pgvector)
- Observability stack (Phoenix/Grafana)

**Helm charts for production** — Structure:
```
agent-system/
  Chart.yaml
  values.yaml
  templates/
    nat-deployment.yaml      # NAT orchestration pods
    nat-service.yaml         # Service + Ingress
    nim-deployment.yaml      # NIM inference pods
    nim-service.yaml         # Internal service (not exposed)
    vectordb-statefulset.yaml
    configmap.yaml           # Guardrails configs, prompts
    secrets.yaml             # API keys, credentials
    hpa.yaml                 # Horizontal Pod Autoscaler
```

Resource planning:
| Component | CPU | Memory | GPU | Notes |
|-----------|-----|--------|-----|-------|
| NAT Server | 4 cores | 8 GB | 0 | CPU-only orchestration |
| NIM (7B model) | 4 cores | 16 GB | 1x A100/H100 | Single GPU sufficient |
| NIM (70B model) | 8 cores | 32 GB | 2x H100 80GB | Tensor parallelism |
| Vector DB | 4 cores | 16 GB | 0 | SSD-backed storage |

### [ADVANCED] Framework-Agnostic Deployment Concepts

**Load balancing for agent endpoints**: Standard round-robin fails for agents because requests have vastly different execution times. Use least-connections or weighted load balancing. For WebSocket connections, use sticky sessions.

**Horizontal vs. vertical scaling**: NAT servers (CPU) scale horizontally easily — add more replicas. NIM servers (GPU) scale vertically first (larger GPUs, tensor parallelism) then horizontally (more GPU nodes). The constraint is usually GPU availability, not NAT server capacity.

**Health checks and readiness probes**:
- Liveness: "Is the process alive?" — restart if not
- Readiness: "Can this instance accept traffic?" — remove from load balancer if not
- For NIM: readiness should verify the model is loaded into GPU memory (can take minutes on cold start)

**Rate limiting and backpressure**: Agent endpoints need token-aware rate limiting, not just request-count limiting. A single complex request may consume 100K tokens. Implement backpressure by returning HTTP 429 with retry-after headers when GPU inference queues are full.

**Blue-green and canary deployments**: Deploy new agent versions alongside old ones. Route a percentage of traffic to the new version. Monitor success rate and latency. This is especially important for prompt changes, which can have unpredictable effects.

### [ADVANCED] Scaling Patterns Specific to Agent Systems

**Variable-length execution**: A simple question might resolve in 1 LLM call (2 seconds). A complex research task might require 15 tool calls and 8 LLM calls (90 seconds). Autoscaling based on request count will underestimate load. Instead, scale on:
- Active agent execution count (in-flight requests)
- GPU inference queue depth
- Token throughput (tokens/second consumed vs. available)

**Token-budget-aware scaling**: If your agent has a 50K token budget per request and you're serving 100 concurrent requests, you need inference capacity for 5M tokens in-flight. Scale NIM replicas based on token throughput, not request count.

**Multi-model scaling**: An agent that uses GPT-4-class reasoning + a small classifier + an embedding model needs independent scaling for each. The reasoning model is the bottleneck; the classifier is cheap. Don't over-provision the cheap models just because the expensive one needs more capacity.

```
Full Deployment Architecture
==========================================================

                    Load Balancer
                         |
            +------------+------------+
            |            |            |
     NAT Server 1  NAT Server 2  NAT Server 3
     (CPU pods)    (CPU pods)    (CPU pods)
            |            |            |
            +-----+------+------+----+
                  |              |
           NIM Cluster      NIM Cluster
           (Reasoning)      (Embedding)
           4x H100          2x A100
                  |              |
                  +------+-------+
                         |
                   Vector Database
                   (StatefulSet)
```

---

## F. Terminology Box

| Term | Definition |
|------|-----------|
| **NIM** | NVIDIA Inference Microservices — containerized, GPU-optimized inference servers for LLMs and other models |
| **NIM Operator** | Kubernetes operator that automates NIM deployment, scaling, and lifecycle management |
| **NVIDIA Dynamo** | Inference acceleration framework that optimizes token generation throughput |
| **NAT Deployment Server** | Frontend server that exposes a NAT agent via a specific protocol (FastAPI, MCP, A2A, etc.) |
| **Async Job** | A long-running agent execution submitted to a background queue, polled by job ID |
| **Dask** | Distributed task scheduler integrated with NAT for async job management |
| **MCP (Model Context Protocol)** | Standard protocol allowing agents to expose capabilities as tools to other agents |
| **A2A (Agent-to-Agent)** | Protocol for inter-agent communication where agents run as separate services |
| **Tensor Parallelism** | Splitting a model across multiple GPUs to serve models larger than single-GPU memory |
| **Backpressure** | Mechanism where a system signals upstream that it cannot accept more work (e.g., HTTP 429) |
| **Readiness Probe** | K8s health check that determines whether a pod can accept traffic |
| **Blue-Green Deployment** | Running two identical environments and switching traffic between them |
| **Canary Deployment** | Routing a small percentage of traffic to a new version before full rollout |
| **Helm Chart** | Kubernetes package manager template for deploying complex multi-component applications |

---

## G. Common Misconceptions

1. **"Deploy the agent as a single container."** Agent systems are multi-container by nature. The orchestrator (NAT), inference server (NIM), vector database, and observability stack are separate concerns with different scaling characteristics. Collapsing them into one container makes scaling impossible.

2. **"Autoscale on CPU utilization."** NAT servers are CPU-bound but are rarely the bottleneck. The bottleneck is GPU inference throughput. Autoscaling on CPU of the NAT pod will not help when NIM is saturated. Scale on inference queue depth or token throughput.

3. **"NIM and NAT are the same thing."** NAT is the orchestration layer (agent logic, tool routing, guardrails). NIM is the inference layer (model serving). NAT calls NIM. They have different resource requirements, scaling characteristics, and deployment patterns.

4. **"WebSocket is always better than REST for agents."** WebSocket is better for streaming responses. But for fire-and-forget async jobs, REST with polling is simpler, more cacheable, and works through more proxies. Choose based on the UX requirement.

5. **"GPU requirements are fixed per model."** GPU requirements depend on batch size, sequence length, quantization, and concurrent request count. A 70B model serving 1 request needs 2x H100. Serving 50 concurrent requests may need 8x H100. Always benchmark under realistic load.

6. **"You can scale agents like stateless microservices."** Agent executions carry state (conversation history, intermediate tool results). If you kill a pod mid-execution, the entire multi-step agent run is lost. Implement graceful shutdown with execution draining.

---

## H. Failure Modes / Anti-Patterns

1. **Cold-start blindness**: NIM containers take 2-10 minutes to load models into GPU memory. If your autoscaler launches new NIM pods in response to a traffic spike, those pods are useless for minutes. Mitigation: maintain warm spare capacity; pre-warm replicas during low-traffic periods.

2. **Shared GPU memory pressure**: Running multiple NIM instances on the same GPU without memory limits causes OOM kills. Always set explicit GPU memory limits and use Kubernetes resource requests to prevent oversubscription.

3. **Synchronous HTTP for long-running agents**: Using synchronous HTTP calls for agents that run 30+ seconds causes client timeouts, load balancer disconnects, and retry storms. Use async job submission or WebSocket streaming.

4. **No backpressure on tool calls**: An agent that calls an external API tool 1000 times during a single execution can DDoS the external service. Implement per-tool rate limits in the NAT configuration.

5. **Monolithic Helm chart**: Packaging NAT, NIM, vector DB, and observability into a single Helm chart prevents independent scaling and versioning. Use separate charts or subcharts with clear interfaces.

6. **Ignoring inference cost in scaling decisions**: Autoscaling that adds GPU nodes in response to traffic spikes can generate massive cloud bills. Implement cost-aware autoscaling with hard budget limits and queue-based admission control.

---

## I. Hands-On Lab

**Lab 10: Deploy a NAT Agent with NIM Backend**

Deploy a multi-step agent using NAT FastAPI Server backed by a NIM inference container. Configure async job management. Implement health checks. Load-test with 10 concurrent users. Observe behavior under load (latency degradation, queue depth). Implement basic backpressure with HTTP 429 responses.

Environment: Docker Compose on a GPU-equipped machine (or cloud GPU instance).

---

## J. Stretch Lab

**Stretch Lab 10: Kubernetes Deployment with Autoscaling**

Deploy the same agent system on Kubernetes using Helm charts. Configure NIM Operator for the inference backend. Implement Horizontal Pod Autoscaler based on custom metrics (inference queue depth). Perform a canary deployment of an updated agent prompt. Verify zero-downtime rollout. Measure cold-start impact on p99 latency.

---

## K. Review Quiz

**Q1.** You have a NAT agent that needs to be callable as a tool by other MCP-compatible agents. Which deployment server should you use?
- A) FastAPI Server
- B) MCP Server
- C) A2A Server
- D) Console Frontend

**Answer: B** — MCP Server exposes the agent as an MCP-compatible tool.

**Q2.** Why is autoscaling on CPU utilization a poor strategy for agent systems?
- A) Agents don't use CPU
- B) The bottleneck is typically GPU inference, not CPU orchestration
- C) Kubernetes doesn't support CPU-based autoscaling
- D) CPU utilization is not measurable for Python processes

**Answer: B** — NAT orchestration is CPU-bound but cheap. NIM inference is GPU-bound and is the bottleneck.

**Q3.** What is the minimum documented GPU requirement for the AI-Q Blueprint Docker Compose setup?
- A) 1x A100 40GB
- B) 1x H100 80GB
- C) 2x H100 80GB
- D) 4x A100 80GB

**Answer: C** — The AI-Q Blueprint documents 2x H100 80GB minimum (or 3x A100 80GB alternative). Always verify hardware requirements on the specific Blueprint page or repository before attempting full deployment. Requirements vary significantly across Blueprints.

**Q4.** A client submits a complex research task that takes 90 seconds. What is the correct NAT pattern?
- A) Synchronous REST call with 120-second timeout
- B) Async job submission, return job ID, client polls for completion
- C) WebSocket connection with synchronous blocking
- D) Retry the request until a faster response is returned

**Answer: B** — Async job management avoids timeout issues and enables proper queue management.

**Q5.** What is the relationship between NAT and NIM in a deployed agent system?
- A) NIM contains NAT as a subcomponent
- B) NAT contains NIM as a subcomponent
- C) NAT handles orchestration and calls NIM for LLM inference
- D) They are alternative choices for the same function

**Answer: C** — NAT is the orchestration layer; NIM is the inference layer. NAT calls NIM.

**Q6.** Which deployment server enables multi-agent systems where each agent runs as an independent service?
- A) FastAPI Server
- B) MCP Server
- C) FastMCP Server
- D) A2A Server

**Answer: D** — A2A Server supports Agent-to-Agent protocol for distributed multi-agent execution.

**Q7.** Why do NIM readiness probes need special consideration compared to standard web services?
- A) NIM doesn't support health checks
- B) Model loading into GPU memory can take several minutes
- C) NIM only responds to gRPC health checks
- D) Readiness probes interfere with inference

**Answer: B** — NIM containers need time to load models into GPU. A pod may be running but not ready to serve inference.

**Q8.** What does NVIDIA Dynamo provide for agent systems?
- A) Agent orchestration logic
- B) Inference acceleration and throughput optimization
- C) Guardrail enforcement
- D) Tool registry management

**Answer: B** — Dynamo accelerates inference throughput, reducing per-step LLM latency.

**Q9.** An agent uses a 70B reasoning model and a small embedding model. How should scaling be approached?
- A) Scale both models identically
- B) Scale them independently — the reasoning model needs more GPU resources
- C) Only scale the embedding model since it handles more requests
- D) Put both models on the same GPU to save resources

**Answer: B** — Different models have different resource needs and bottleneck characteristics. Scale independently.

**Q10.** What is the correct Dask integration pattern in NAT?
- A) Dask replaces the LLM for inference
- B) Dask manages async job queuing and execution scheduling
- C) Dask provides the vector database backend
- D) Dask handles WebSocket connection management

**Answer: B** — Dask integrates with NAT for async job management — queuing, scheduling, and status tracking.

---

## L. Mini Project

**Design a production deployment plan for a customer-support agent system.**

Requirements: 100 concurrent users, sub-5-second response for simple queries, sub-60-second for complex research, 99.9% uptime SLA.

Deliverables:
1. Architecture diagram showing all components (NAT, NIM, vector DB, load balancer)
2. GPU resource estimate with justification
3. Helm chart `values.yaml` with resource requests, replica counts, and HPA configuration
4. Async job strategy for long-running queries
5. Health check and readiness probe configuration
6. Scaling strategy document explaining how auto-scaling handles traffic spikes

---

## M. How This May Appear on the Exam

1. **Scenario-based server selection**: "An agent needs to be accessible to other agents in an MCP ecosystem. Which NAT deployment server is appropriate?" Expect 2-3 questions requiring you to match use cases to deployment server types.

2. **NAT vs. NIM architecture**: "Which component handles agent orchestration logic vs. LLM inference?" The exam will test whether you understand the separation of concerns.

3. **Resource estimation**: "Given a 70B parameter model and a target of 50 concurrent users, which GPU configuration is sufficient?" Expect questions about hardware requirements, especially referencing documented Blueprint configurations.

4. **Scaling strategy**: "Why is request-count-based autoscaling insufficient for agent endpoints?" Expect questions about variable-length execution and token-aware scaling.

5. **Async patterns**: "A user submits a complex query. What is the appropriate response pattern?" Expect questions testing async job management knowledge.

---

## N. Checklist for Mastery

- [ ] I can list all five NAT deployment servers and explain when to use each one
- [ ] I can explain the architectural separation between NAT (orchestration) and NIM (inference)
- [ ] I can configure NAT async job management with Dask and explain the submission/polling pattern
- [ ] I can deploy a NIM container with Docker and configure GPU resource allocation
- [ ] I can describe what NIM Operator provides for Kubernetes deployments
- [ ] I can estimate GPU requirements for an agent workload given model size and concurrency targets
- [ ] I can design a Helm chart structure for a multi-component agent deployment
- [ ] I can explain why agent systems need different autoscaling strategies than stateless APIs
- [ ] I can implement health checks and readiness probes for both NAT and NIM containers
- [ ] I can configure backpressure mechanisms (HTTP 429, queue limits) for agent endpoints
- [ ] I can describe blue-green and canary deployment strategies for agent prompt updates
- [ ] I can reference the AI-Q Blueprint hardware requirements (2x H100 80GB minimum). Always verify hardware requirements on the specific Blueprint page or repository before attempting full deployment. Requirements vary significantly across Blueprints.
- [ ] I can explain the cold-start problem for NIM containers and mitigation strategies
- [ ] I can design a multi-model scaling strategy where reasoning and embedding models scale independently

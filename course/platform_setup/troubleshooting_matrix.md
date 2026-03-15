# Troubleshooting Matrix

Common issues students hit during this course, organized by symptom. For each issue: what you see, why it happens, and how to fix it.

---

## Authentication and API Issues

### 1. `401 Unauthorized` from NIM API

- **Symptom:** API calls return `401 Unauthorized`.
- **Likely cause:** NVIDIA_API_KEY not set, expired, or malformed.
- **Fix:** Regenerate your key at [build.nvidia.com](https://build.nvidia.com). Verify it is set with `echo $NVIDIA_API_KEY`. Ensure there is no trailing whitespace or newline in the value.
- **Affected:** All modules and labs using NIM.

### 2. `404 Not Found` from NIM endpoint

- **Symptom:** API calls return `404 Not Found`.
- **Likely cause:** Wrong model name or wrong base URL.
- **Fix:** Check the exact model name in the [build.nvidia.com API catalog](https://build.nvidia.com). Use `integrate.api.nvidia.com` as the base URL, not `api.nvidia.com`.
- **Affected:** All modules and labs using NIM.

### 3. `429 Too Many Requests`

- **Symptom:** API calls return `429 Too Many Requests`.
- **Likely cause:** Free tier rate limit hit.
- **Fix:** Wait and retry. Reduce request frequency in your code. Consider a paid tier for high-volume labs.
- **Affected:** Labs 4, 8 (high volume).

### 4. NGC container pull fails with authentication error

- **Symptom:** `docker pull nvcr.io/nvidia/...` fails with an authentication error.
- **Likely cause:** NGC API key not configured or expired.
- **Fix:** Generate an NGC API key at [ngc.nvidia.com](https://ngc.nvidia.com). Run `docker login nvcr.io` with username `$oauthtoken` and your NGC API key as the password.
- **Affected:** platform_setup/optional_self_hosted_nim.md (optional).

---

## NAT Installation and Import Issues

### 5. `ModuleNotFoundError: No module named 'nat'`

- **Symptom:** Python raises `ModuleNotFoundError` when importing NAT.
- **Likely cause:** `nvidia-nat` not installed, or you are in the wrong Python environment.
- **Fix:** Run `pip install nvidia-nat`. Verify your environment with `which python` and make sure it matches where you installed the package.
- **Affected:** All NAT labs.

### 6. `ModuleNotFoundError` for NAT eval features

- **Symptom:** Import error when trying to use evaluation features (e.g., `nat.eval` or similar).
- **Likely cause:** Evaluation extras not installed. The base `nvidia-nat` package may not include the eval framework.
- **Fix:** Run `pip install nvidia-nat[eval]` or `pip install nvidia-nat-eval` (check current docs for the correct package/extra name).
- **Affected:** M8, Labs 8-9.

### 7. NAT Profiler command not found or import error

- **Symptom:** Profiler command is not recognized or import fails.
- **Likely cause:** Profiling extras not installed.
- **Fix:** Run `pip install nvidia-nat[profiling]`, or check if profiling is part of the eval extra.
- **Affected:** M8, Labs 2, 6, 8.

### 8. NAT LangChain plugin import error

- **Symptom:** Import error when using NAT with LangChain.
- **Likely cause:** LangChain extras not installed.
- **Fix:** Run `pip install nvidia-nat[langchain] langchain langchain-nvidia-ai-endpoints`.
- **Affected:** M6, Lab 6.

---

## NAT Configuration and Runtime Issues

### 9. NAT YAML config error -- "invalid key" or "unknown field"

- **Symptom:** NAT raises a config error when loading a YAML workflow file.
- **Likely cause:** YAML syntax error, wrong indentation, or field name that does not match the current NAT version.
- **Fix:** Validate your YAML with `python -c "import yaml; yaml.safe_load(open('workflow.yaml'))"`. Check the NAT docs for correct field names in your version.
- **Affected:** All NAT labs.

### 10. NAT agent runs but produces no tool calls

- **Symptom:** The agent completes a run but never invokes any tools.
- **Likely cause:** Tool/function not registered correctly, or the system prompt does not instruct tool use.
- **Fix:** Verify function registration. Check that your agent type supports tool calling (not all do). Add an explicit tool-use instruction to the system prompt.
- **Affected:** Labs 1-2, 5.

---

## NeMo Guardrails Issues

### 11. `nemoguardrails chat` hangs or errors on startup

- **Symptom:** The interactive chat command hangs or prints an error immediately.
- **Likely cause:** The model name in `config.yml` does not match an available model, `NVIDIA_API_KEY` is not set, `langchain-nvidia-ai-endpoints` is not installed, or `colang_version: "2.x"` is missing when using Colang 2 syntax.
- **Fix:** (1) Verify the model name matches a model at [build.nvidia.com](https://build.nvidia.com). (2) Ensure `NVIDIA_API_KEY` is exported. (3) Ensure `langchain-nvidia-ai-endpoints` is installed — without it, `engine: nim` and `engine: nvidia_ai_endpoints` cannot reach NVIDIA-hosted models. (4) Ensure `colang_version: "2.x"` is set in `config.yml` if your `.co` files use Colang 2 syntax.
- **Affected:** M9, Lab 9, platform_setup/first_guardrails_run.md.

### 11b. `nemoguardrails server` fails with ImportError (FastAPI/uvicorn missing)

- **Symptom:** Running `nemoguardrails server` fails with `ModuleNotFoundError: No module named 'fastapi'` or `No module named 'uvicorn'`, or the `server` subcommand is not recognized.
- **Likely cause:** The base `nemoguardrails` package does not include server dependencies. Server mode requires an optional extra.
- **Fix:** Install server dependencies: `pip install nemoguardrails[server]`. If the `[server]` extra is not recognized in your version, check the NeMo Guardrails release notes for the correct extra name, or install manually: `pip install fastapi uvicorn`.
- **Affected:** M9 (Step 10 of first_guardrails_run), Lab 9, M10.

### 12. Colang flow does not trigger (messages pass through unguarded)

- **Symptom:** Messages that should be blocked or redirected go through without triggering any rail.
- **Likely cause:** Flow definition is too narrow, a missing `activate` statement, or a Colang version mismatch.
- **Fix:** Test with exact phrases first. Ensure you are using Colang 2.0 syntax if your `config.yml` specifies `colang_version: "2.x"`. Check the `config.yml` for the `colang_version` setting.
- **Affected:** M9, Lab 9.

### 13. Colang blocks EVERYTHING including valid messages

- **Symptom:** All user messages are blocked or refused, even clearly valid ones.
- **Likely cause:** Flow definition is too broad, or the default action is set to block.
- **Fix:** Narrow the flow conditions. Test with known-good inputs first. Use `nemoguardrails evaluate` to measure precision.
- **Affected:** M9, Lab 9.

---

## Observability Issues

### 14. Phoenix UI shows no traces (empty dashboard)

- **Symptom:** You open the Phoenix UI but see no trace data.
- **Likely cause:** NAT observability is not configured to export to Phoenix, Phoenix server is not running, or the port is wrong.
- **Fix:** Verify Phoenix is running at `localhost:6006`. Verify that your NAT observability config points to the correct endpoint. Run a query and wait 5-10 seconds for trace propagation.
- **Affected:** M11, Lab 11.

### 15. Phoenix traces show but are missing spans (e.g., no tool call spans)

- **Symptom:** Traces appear in Phoenix but some expected spans (like tool calls) are missing.
- **Likely cause:** Tool calls are not instrumented, or span export is batched and has not flushed yet.
- **Fix:** Force flush with an explicit shutdown call. Verify that your NAT observability depth config covers tool-level tracing.
- **Affected:** M11, Lab 11.

---

## Infrastructure Issues

### 16. Redis connection refused

- **Symptom:** `ConnectionRefusedError` or similar when connecting to Redis.
- **Likely cause:** Redis server is not running, the port is wrong, or Docker is not started.
- **Fix:** Start Redis with `docker run -d -p 6379:6379 redis` and verify with `redis-cli ping` (should return `PONG`).
- **Affected:** M3, Lab 3, Lab 10.

### 17. Docker Compose fails to start agent service

- **Symptom:** `docker compose up` fails or the service exits immediately.
- **Likely cause:** Missing environment variables, port conflicts, or image not built.
- **Fix:** Check logs with `docker compose logs <service>`. Ensure your `.env` file exists and contains `NVIDIA_API_KEY`. Check for port conflicts with `lsof -i :<port>`.
- **Affected:** Lab 10, Capstone 1.

### 18. Self-hosted NIM container runs but inference is extremely slow

- **Symptom:** NIM responds but each request takes minutes instead of seconds.
- **Likely cause:** GPU not detected by the container, or insufficient GPU memory for the model.
- **Fix:** Verify `nvidia-smi` works inside the container. Check that `nvidia-container-toolkit` is installed on the host. Use a smaller model if GPU memory is insufficient.
- **Affected:** platform_setup/optional_self_hosted_nim.md (optional).

---

## RAG and Retrieval Issues

### 19. Embedding endpoint returns wrong dimensionality

- **Symptom:** Vector dimension mismatch error when indexing or querying.
- **Likely cause:** Using the wrong embedding model or expecting a different vector size.
- **Fix:** Check the model card on [build.nvidia.com](https://build.nvidia.com) for the output dimension of your embedding model. Ensure your FAISS or Milvus index dimension matches.
- **Affected:** M4, Lab 4.

### 20. FAISS search returns irrelevant results

- **Symptom:** Retrieved documents are not relevant to the query.
- **Likely cause:** Embedding model mismatch between indexing and querying, chunks too large or too small, or no reranking step.
- **Fix:** Verify you are using the same embedding model for both indexing and querying. Try different chunk sizes. Add a reranking step after retrieval.
- **Affected:** M4, Lab 4.

---

## Deployment and Server Issues

### 21. NAT interactive workflow WebSocket connection fails

- **Symptom:** WebSocket connection is refused or times out.
- **Likely cause:** Server not started in interactive mode, wrong WebSocket URL, or a firewall blocking the connection.
- **Fix:** Start the NAT server with WebSocket support enabled. Use `ws://localhost:<port>` not `http://`. Check firewall rules if running on a remote machine.
- **Affected:** M12, Lab 12.

### 22. Blueprint Docker Compose requires GPUs you do not have

- **Symptom:** Blueprint deployment fails because it expects GPU resources that are not available on your machine.
- **Likely cause:** You are attempting a full Blueprint deployment without the required hardware.
- **Fix:** Do NOT attempt full deployment without the hardware listed on the Blueprint page. Use the "study + partial reproduction" path from `platform_setup/blueprint_walkthrough.md` instead -- that path works on a CPU laptop with hosted APIs.
- **Affected:** platform_setup/blueprint_walkthrough.md.

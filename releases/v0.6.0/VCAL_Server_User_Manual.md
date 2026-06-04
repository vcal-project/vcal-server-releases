# VCAL Server — User Manual

**VCAL Server** is a production-ready semantic cache for LLM and RAG workloads.  
It reduces repeated LLM calls, cuts token costs, and serves cached answers with millisecond latency while running inside your infrastructure.

VCAL follows an **open-core model**:

- **vcal-core** — open-source Rust caching engine
- **VCAL Server** — commercial HTTP service with licensing, observability, persistence, and deployment tooling

---

## 1. What VCAL Server Does

VCAL Server acts as a **semantic cache layer** for LLM applications.

Your application remains responsible for:

1. embedding the user input into a vector,
2. asking VCAL Server whether a similar answer already exists,
3. calling the LLM on a cache miss,
4. storing the new vector and answer back into VCAL Server.

VCAL Server itself does **not** call the LLM and does **not** generate embeddings automatically.

Typical use cases:

- RAG pipelines such as LangChain, LlamaIndex, and custom retrieval systems
- Chatbots and assistants
- High-traffic LLM backends
- Token cost and latency reduction in production systems
- On-prem or VPC AI workloads where cached answers must stay inside the customer perimeter

---

## 2. System Requirements

### Supported platforms

- Linux x86_64 (glibc)
- Linux x86_64 (musl / static)
- Linux aarch64 (musl / static)
- Docker (recommended)

### Minimum resources

- CPU: 1 core
- RAM: 512 MB minimum, 1–2 GB recommended for small evaluations
- Disk: depends on cache size; start with 1–5 GB for trials

For larger production caches, size memory and disk according to the number of vectors, vector dimensions, retained answers, TTL, and snapshot strategy.

---

## 3. Quickstart (Docker — recommended)

The fastest way to get started is Docker with a persisted data directory.

### 3.1 Create a data directory

```bash
mkdir -p ./vcal-data
```

This directory stores:

- the license file
- persistent cache data and snapshots, depending on your configuration

On Linux, if the container cannot write to the mounted directory, set ownership for the container user:

```bash
sudo chown -R 10001:10001 ./vcal-data
```

---

### 3.2 Request a 30-day trial license

Run the license command inside the container:

```bash
docker run --rm -it \
  -v "$(pwd)/vcal-data:/var/lib/vcal" \
  -e VCAL_LICENSE_PATH=/var/lib/vcal/license.json \
  ghcr.io/vcal-project/vcal-server:v0.6.0 \
  license trial <your_email>
```

You will receive a verification code by email.

---

### 3.3 Verify the license

```bash
docker run --rm -it \
  -v "$(pwd)/vcal-data:/var/lib/vcal" \
  -e VCAL_LICENSE_PATH=/var/lib/vcal/license.json \
  ghcr.io/vcal-project/vcal-server:v0.6.0 \
  license verify <code>
```

This writes a signed license file to:

```text
./vcal-data/license.json
```

---

### 3.4 Start VCAL Server

```bash
docker run --rm -p 8080:8080 \
  -v "$(pwd)/vcal-data:/var/lib/vcal" \
  -e VCAL_LICENSE_PATH=/var/lib/vcal/license.json \
  -e VCAL_DIMS=768 \
  -e VCAL_CAP_MAX_BYTES=8589934592 \
  -e RUST_LOG=info \
  ghcr.io/vcal-project/vcal-server:v0.6.0
```

VCAL Server is now available at:

```text
http://localhost:8080
```

---

### 3.5 Test the server

After starting VCAL Server, verify that it is running correctly.

#### 1) Health and readiness checks

These endpoints require no request body:

```bash
curl -fsS http://localhost:8080/healthz && echo "health OK"
curl -fsS http://localhost:8080/readyz && echo "ready OK"
```

A healthy server returns HTTP 200 OK.

#### 2) Test `/v1/qa` with a schema-correct vector

The `/v1/qa` endpoint operates on embedding vectors, not raw text. The `query` array length must match `VCAL_DIMS`, for example 768.

If authentication is enabled, include your application API key through `X-VCAL-Key` or the configured Bearer token mechanism.

```bash
curl -s http://localhost:8080/v1/qa \
  -H 'Content-Type: application/json' \
  -H 'X-VCAL-Key: <your_app_key>' \
  -d "$(python3 - <<'PY'
import json

dims = 768  # must match VCAL_DIMS
print(json.dumps({
    "query": [0.0] * dims,
    "k": 1,
    "ef": 128,
    "sim_threshold": 0.8
}))
PY
)"
```

Expected behavior on an empty cache:

- HTTP 200 OK
- response contains `"hit": false`
- no vector dimension error

This confirms that:

- the server is reachable,
- the API contract is respected,
- vector dimensionality matches the configured index dimensions.

#### 3) Common errors and fixes

| Error | Cause | Fix |
| --- | --- | --- |
| `400 Bad Request: dimension mismatch` | Vector length does not match `VCAL_DIMS` | Send vectors with the configured dimension |
| `401 Unauthorized` | Authentication is enabled but no valid key was sent | Add `X-VCAL-Key` or Bearer auth |
| `404 Not Found` | Wrong endpoint | Use `/v1/qa`, `/v1/search`, `/v1/upsert`, `/healthz`, or `/readyz` |
| `connection refused` | Server is not running or port is wrong | Check container logs and port mapping |

---

## 4. Binary Installation (Linux)

### 4.1 Download

Download the appropriate archive for your platform from: https://vcal-project.com/vcal-server/ -> Download -> Binaries

Each binary release includes:

- `.tar.gz` archive
- `.sha256` checksum
- `.minisig` signature

The latest release metadata is available at:

```text
https://downloads.vcal-project.com/vcal-server/latest.json
```

---

### 4.2 Verify integrity

Verify the checksum:

```bash
sha256sum -c vcal-server-*.sha256
```

If you use Minisign, verify the signature with the published public key.

---

### 4.3 Install and run

```bash
tar -xzf vcal-server-*.tar.gz
chmod +x vcal-server
```

Optional system-wide install:

```bash
sudo install -m 0755 vcal-server /usr/local/bin/vcal-server
```

---

### 4.4 Trial license for binary installation

For a user-writable license path:

```bash
mkdir -p "$HOME/.vcal"
export VCAL_LICENSE_PATH="$HOME/.vcal/license.json"
vcal-server license trial <your_email>
vcal-server license verify <code>
vcal-server
```

For a system-wide license path, use `/etc/vcal/license.json`. This normally requires `sudo`:

```bash
sudo mkdir -p /etc/vcal
sudo VCAL_LICENSE_PATH=/etc/vcal/license.json vcal-server license trial <your_email>
sudo VCAL_LICENSE_PATH=/etc/vcal/license.json vcal-server license verify <code>
VCAL_LICENSE_PATH=/etc/vcal/license.json vcal-server
```

---

## 5. Configuration

VCAL Server is configured primarily through environment variables.

Common options:

```bash
VCAL_DIMS=768
VCAL_CAP_MAX_VECTORS=1000000
VCAL_CAP_MAX_BYTES=8589934592
VCAL_TTL_SECS=2592000
VCAL_AUTOSAVE_SECS=3600
VCAL_AUTOSAVE_ATOMIC=1
VCAL_EVICT_INTERVAL_SECS=3600
VCAL_LICENSE_PATH=/var/lib/vcal/license.json
RUST_LOG=info
```

Authentication-related settings depend on your deployment, but production deployments should normally configure separate application and administration keys.

If `VCAL_DIMS` is not recoverable from an existing snapshot, set it explicitly. The vector dimension must match the embedding model used by your application.

---

## 6. Licensing Model

- **Trial (30-day evaluation)**  
  Full-feature, single-instance license issued through the CLI.

- **Growth (annual license)**  
  Production use for one application or service.

- **Enterprise**  
  Multi-app, security reviews, SLAs, and custom terms.

VCAL Server will not start without a valid license file.

---

## 7. API Overview

VCAL Server exposes a JSON HTTP API.

Common endpoints:

- `GET  /healthz` — liveness check
- `GET  /readyz` — readiness check
- `GET  /metrics` — Prometheus metrics
- `POST /v1/qa` — semantic question/answer cache lookup
- `POST /v1/search` — vector similarity lookup
- `POST /v1/batch_search` — batch vector similarity lookup
- `POST /v1/insert` — insert a new vector or answer
- `POST /v1/upsert` — insert or update a vector and answer
- `POST /v1/delete` — delete a cached item
- `POST /v1/snapshot/save` — save a snapshot
- `POST /v1/answers/save` — save answer storage
- `POST /v1/answers/load` — load answer storage
- `POST /v1/decision` — cache decision endpoint, if enabled in your build/configuration

Full API documentation:

```text
https://server.docs.vcal-project.com
```

### OpenAPI and Docs UI Availability

The static OpenAPI specification is provided as a release artifact where applicable.

The standard free testing build of VCAL Server does not expose the built-in OpenAPI/docs UI at runtime. The docs UI is disabled by default in the binary build.

For normal use, refer to the public documentation and the static `openapi.yml` file: https://server.docs.vcal-project.com/

Customers who need runtime OpenAPI/docs UI access can request a custom build through VCAL Server support.

---

## 8. Python Integration Example

VCAL Server integrates as a lightweight vector cache in front of your LLM. Your application is responsible for embedding text into vectors and calling the LLM on cache misses.

The example below uses Ollama embeddings by default. Replace the embedding function and LLM fallback with your production providers.

```python
#!/usr/bin/env python3
"""
Minimal VCAL integration flow:

1. Embed the user question.
2. Ask VCAL Server /v1/qa with the embedding vector.
3. On hit, return the cached answer.
4. On miss, call your LLM and store the answer via /v1/upsert.
"""

import hashlib
import os
import requests

VCAL_BASE = os.getenv("VCAL_URL", "http://127.0.0.1:8080").rstrip("/")
VCAL_KEY = os.getenv("VCAL_API_KEY", "")

OLLAMA_URL = os.getenv("OLLAMA_URL", "http://127.0.0.1:11434").rstrip("/")
EMBED_MODEL = os.getenv("EMBED_MODEL", "nomic-embed-text")

SIM_THRESHOLD = float(os.getenv("VCAL_SIM_THR", "0.85"))
EF_SEARCH = int(os.getenv("VCAL_EF_SEARCH", "128"))

HEADERS = {"Content-Type": "application/json"}
if VCAL_KEY:
    HEADERS["X-VCAL-Key"] = VCAL_KEY


def embed_text(text: str) -> list[float]:
    r = requests.post(
        f"{OLLAMA_URL}/api/embeddings",
        json={"model": EMBED_MODEL, "prompt": text},
        timeout=30,
    )
    r.raise_for_status()
    vec = r.json().get("embedding")
    if not isinstance(vec, list) or not vec:
        raise RuntimeError("Bad embedding response")
    return vec


def ext_id_for(question: str) -> int:
    normalized = question.strip().lower().encode("utf-8")
    digest = hashlib.blake2b(normalized, digest_size=8).digest()
    return int.from_bytes(digest, "big", signed=False)


def vcal_qa(vec: list[float]) -> dict:
    r = requests.post(
        f"{VCAL_BASE}/v1/qa",
        headers=HEADERS,
        json={
            "query": vec,
            "k": 1,
            "ef": EF_SEARCH,
            "sim_threshold": SIM_THRESHOLD,
        },
        timeout=30,
    )
    r.raise_for_status()
    return r.json()


def vcal_upsert(ext_id: int, vec: list[float], answer: str) -> None:
    r = requests.post(
        f"{VCAL_BASE}/v1/upsert",
        headers=HEADERS,
        json={
            "ext_id": ext_id,
            "vector": vec,
            "answer": answer,
        },
        timeout=30,
    )
    r.raise_for_status()


def call_llm_fallback(question: str) -> str:
    # Replace this placeholder with OpenAI, Ollama, Anthropic, Mistral,
    # your internal model gateway, or your RAG pipeline.
    return f"LLM-generated answer for: {question}"


def main() -> None:
    question = input("Ask a question: ").strip()
    if not question:
        return

    vec = embed_text(question)
    qa = vcal_qa(vec)

    if qa.get("hit") and qa.get("answer"):
        print("VCAL HIT")
        print(qa["answer"])
        return

    print("VCAL MISS — calling LLM fallback")
    answer = call_llm_fallback(question)
    vcal_upsert(ext_id_for(question), vec, answer)
    print("Stored in VCAL")
    print(answer)


if __name__ == "__main__":
    main()
```

Production recommendations:

- use a real embedding model and keep `VCAL_DIMS` aligned with that model,
- set authentication headers,
- add retries and timeouts,
- log cache hit/miss decisions,
- avoid storing sensitive data unless your deployment policy allows it.

---

## 9. Metrics & Monitoring

VCAL Server exports Prometheus-compatible metrics through `/metrics`.

Common metric areas include:

- request counts and errors
- search latency
- cache hits and misses
- tokens saved
- cached answer count
- TTL and LRU evictions
- license validity and days until expiry

A ready-to-import Grafana dashboard JSON is provided alongside the release artifacts.

---

## 10. Data & Persistence

VCAL Server supports persistent cache data and snapshots.

Recommended production practices:

- mount a persistent data directory,
- keep the license file outside ephemeral container storage,
- enable autosave for long-running deployments,
- back up the data directory before upgrades,
- use atomic autosave where supported.

No external database is required for the default deployment model.

---

## 11. Upgrading

Recommended upgrade procedure:

1. Stop the server or drain traffic.
2. Back up the data directory and license file.
3. Replace the binary or Docker image.
4. Start the new version.
5. Check `/healthz`, `/readyz`, and `/metrics`.
6. Run a small `/v1/qa` smoke test with the configured vector dimension.

---

## 12. Security Notes

- Run behind a reverse proxy in production.
- Protect application endpoints with authentication.
- Do not expose `/metrics` publicly without protection.
- Restrict network access where possible.
- Keep license files and API keys out of public repositories.
- Review cached answer content before storing regulated or highly sensitive data.

---

## 13. Support & Documentation

- vcal-core docs: https://vcal-core.docs.vcal-project.com/
- VCAL Server docs: https://server.docs.vcal-project.com
- Support: support@vcal-project.com

---

## 14. License

VCAL Server is distributed under a commercial license.

VCAL Server is developed by VCAL Labs, Inc.

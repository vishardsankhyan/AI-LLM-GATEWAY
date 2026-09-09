# Day 3 — LiteLLM Gateway Runtime: Docker, Networking & Provider Validation

## Learning Headline

> **Build and validate a production-style LiteLLM gateway runtime: Docker → networking → secrets → provider routing → end-to-end LLM request.**

---

## 1. Day 3 Objective

Build a working architecture:

```text
Client
  │
  ▼
LiteLLM Gateway :4000
  │
  ▼
Groq API
  │
  ▼
LLM Response
```

By the end of the session:

* LiteLLM runs inside Docker
* Docker networking works from the container to the internet
* Groq credentials are securely passed through the environment
* LiteLLM correctly routes the logical model to Groq
* `/v1/chat/completions` successfully returns an LLM response

---

# 2. Minimum Steps to Reach the Working State

## Step 1 — Pin the LiteLLM Version

Use a known stable image rather than `latest`.

```yaml
image: ghcr.io/berriai/litellm:1.100.0
```

### Why

Version pinning makes the environment reproducible.

```text
latest
  ↓
version can change unexpectedly

1.100.0
  ↓
reproducible runtime
```

---

# 3. Create the LiteLLM Configuration

`docker/config.yaml`

```yaml
model_list:
  - model_name: gateway-test
    litellm_params:
      model: groq/openai/gpt-oss-120b
      api_key: os.environ/GROQ_API_KEY
```

### Important Concept

The client does **not** need to know that Groq is being used.

The client sends:

```json
{
  "model": "gateway-test"
}
```

LiteLLM translates:

```text
gateway-test
     ↓
groq/openai/gpt-oss-120b
     ↓
Groq API
```

This is the beginning of **provider abstraction**.

---

# 4. Create Docker Compose

`docker/docker-compose.yml`

```yaml
services:
  litellm:
    image: ghcr.io/berriai/litellm:1.100.0

    ports:
      - "4000:4000"

    command:
      - "--config"
      - "/app/config.yaml"
      - "--port"
      - "4000"

    environment:
      - GROQ_API_KEY

    dns:
      - 8.8.8.8
      - 1.1.1.1

    volumes:
      - ${PWD}/docker/config.yaml:/app/config.yaml:ro
```

### Important Points

```yaml
environment:
  - GROQ_API_KEY
```

passes the Codespaces environment variable into the container.

Without this, the host may have the secret while the container does not.

---

# 5. Recreate the Container

```bash
docker compose -f docker/docker-compose.yml up -d --force-recreate
```

Check:

```bash
docker compose -f docker/docker-compose.yml ps
```

Expected:

```text
litellm    Up    0.0.0.0:4000->4000/tcp
```

---

# 6. Validate LiteLLM Health

```bash
curl http://localhost:4000/health/liveliness
```

Expected:

```text
i am alive
```

At this point:

```text
Docker
  ↓
LiteLLM
  ↓
Port 4000
```

is working.

---

# 7. Validate the Secret Inside the Container

Do **not** print the actual API key.

Use:

```bash
docker exec docker-litellm-1 sh -c \
'if [ -n "$GROQ_API_KEY" ]; then echo "GROQ_API_KEY is set"; else echo "GROQ_API_KEY is NOT set"; fi'
```

Expected:

```text
GROQ_API_KEY is set
```

### Learning Point

Always distinguish:

```text
Host environment
       ≠
Container environment
```

A secret existing in Codespaces does not automatically mean the application container receives it.

---

# 8. Validate the Complete Gateway Path

```bash
curl -X POST http://localhost:4000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gateway-test",
    "messages": [
      {
        "role": "user",
        "content": "Reply with exactly: LiteLLM to Groq works"
      }
    ]
  }'
```

Successful response confirmed:

```text
LiteLLM to Groq works
```

Therefore:

```text
Client
  │
  ▼
localhost:4000
  │
  ▼
LiteLLM
  │
  ▼
gateway-test
  │
  ▼
groq/openai/gpt-oss-120b
  │
  ▼
Groq API
  │
  ▼
Response
```

---

# 9. Troubleshooting Journey — What We Learned

The successful path above is the **minimum path**.

The following investigation was required because the first end-to-end request failed.

---

## Problem 1 — LiteLLM Could Not Reach Groq

Initial symptom:

```text
Temporary failure in name resolution
```

for:

```text
api.groq.com
```

### Investigation

First verify the host:

```bash
curl -I https://api.groq.com
```

The host could reach Groq.

Therefore:

```text
Host → Internet       ✅
Container → Internet  ❌
```

This isolated the problem to Docker networking rather than Groq.

---

# 10. Test Docker DNS Independently

A temporary Docker container was used to test external connectivity.

The important diagnostic question was:

```text
Can a Docker container resolve and reach an external service?
```

The default Docker networking path was compared with the Compose/custom bridge network.

Result:

```text
Host networking       ✅
Temporary container   ✅
Compose bridge        ❌
```

This pointed toward Docker bridge forwarding rather than DNS configuration alone.

---

# 11. Inspect Docker Network

The custom bridge was inspected:

```bash
docker network inspect ai-gateway-test
```

Important information:

```text
Driver: bridge
Subnet: 172.19.0.0/16
Gateway: 172.19.0.1
```

The container had:

```text
172.19.0.2
```

and a default route through:

```text
172.19.0.1
```

Therefore container attachment and routing existed.

---

# 12. Inspect Host Routing

```bash
ip route
```

The important observation was that Docker bridge traffic had a valid bridge route, while the host itself had internet connectivity.

This reduced the problem further:

```text
Container
   ↓
Docker bridge
   ↓
Host
   ↓
Internet
```

The failure was between the Docker bridge and host forwarding/NAT path.

---

# 13. Check Docker Firewall Backend

```bash
docker info
```

Relevant result:

```text
Firewall Backend: iptables
```

Then:

```bash
iptables --version
iptables-legacy --version
```

The system had both:

```text
iptables-nft
iptables-legacy
```

This was the critical clue.

---

# 14. Compare Firewall Policies

Current nft-compatible rules showed Docker forwarding/NAT rules.

However:

```bash
sudo iptables-legacy -S FORWARD
```

showed:

```text
-P FORWARD DROP
```

This meant the legacy firewall path could still drop forwarded Docker packets.

### Key Architecture Insight

We effectively had two firewall rule worlds:

```text
Docker rules
    │
    ▼
iptables/nftables path
    │
    │
    X   legacy FORWARD policy = DROP
```

Docker's rules alone were not sufficient because the legacy forwarding policy was still dropping traffic.

---

# 15. Definitive Test

The forwarding policy was temporarily changed:

```bash
sudo iptables-legacy -P FORWARD ACCEPT
```

Then the external connectivity test was repeated.

Result:

```text
TCP EXTERNAL = OK
```

This was the decisive test.

It proved:

```text
Docker networking configuration
        +
Docker bridge
        +
NAT
        +
Internet
```

were fundamentally working.

The blocking condition was:

```text
iptables-legacy FORWARD DROP
```

---

# 16. Important Troubleshooting Lesson

Do not immediately assume:

```text
DNS failure = DNS problem
```

A container reporting:

```text
api.groq.com could not be resolved
```

can actually be caused by a broader network forwarding problem.

The troubleshooting sequence was:

```text
Application
   ↓
DNS
   ↓
Container connectivity
   ↓
Docker bridge
   ↓
Host forwarding
   ↓
Firewall
   ↓
Internet
```

We progressively isolated each layer instead of changing random configuration.

---

# 17. Problem 2 — Groq Invalid API Key

After networking was fixed, LiteLLM reached Groq but returned:

```text
Invalid API Key
```

This was an important change.

It meant:

```text
DNS             ✅
TCP connectivity ✅
Groq reachable   ✅
LiteLLM routing  ✅
Authentication  ❌
```

The problem had moved to the credential layer.

---

# 18. Compare Host vs Container Environment

Host:

```text
GROQ_API_KEY is set
```

Container:

```text
GROQ_API_KEY is NOT set
```

Root cause:

The Compose file had not passed the environment variable into the container.

---

# 19. Fix Secret Injection

Added:

```yaml
environment:
  - GROQ_API_KEY
```

Then recreated:

```bash
docker compose -f docker/docker-compose.yml up -d --force-recreate
```

Validated without exposing the secret:

```text
CONTAINER: GROQ_API_KEY is set
```

---

# 20. Final Validation

The same `/v1/chat/completions` request was executed again.

Result:

```text
LiteLLM to Groq works
```

Final status:

| Layer                 | Status |
| --------------------- | ------ |
| Codespaces            | ✅      |
| Docker                | ✅      |
| Docker bridge         | ✅      |
| Host forwarding       | ✅      |
| Internet connectivity | ✅      |
| DNS                   | ✅      |
| LiteLLM               | ✅      |
| Configuration         | ✅      |
| Secret injection      | ✅      |
| Provider routing      | ✅      |
| Groq authentication   | ✅      |
| LLM completion        | ✅      |

---

# 21. Day 3 Key Learnings

### 1. Container networking is a layered system

```text
Application
→ DNS
→ TCP/IP
→ Docker bridge
→ NAT
→ Firewall
→ Internet
```

Troubleshoot from the bottom up.

### 2. Host configuration ≠ container configuration

A secret available in Codespaces must explicitly be made available to the container.

### 3. Provider syntax matters

This:

```yaml
model: openai/gpt-oss-120b
```

selects the OpenAI provider.

This:

```yaml
model: groq/openai/gpt-oss-120b
```

selects Groq.

### 4. Logical model names enable abstraction

The client only sees:

```text
gateway-test
```

The gateway controls the actual provider/model.

### 5. HTTP success is not enough

A healthy gateway:

```text
/health/liveliness → alive
```

does **not** prove:

```text
Gateway → Provider → LLM
```

works.

The real validation is an end-to-end inference request.

---

# 22. Day 3 Completion Criteria

Day 3 is complete when all of these are true:

```text
[✓] LiteLLM container starts
[✓] Port 4000 exposed
[✓] Health endpoint works
[✓] Container has provider credentials
[✓] Container can reach provider
[✓] Provider routing works
[✓] LLM request succeeds
```

## Current State

```text
                    ┌─────────────────┐
                    │     Client      │
                    └────────┬────────┘
                             │
                             │ OpenAI-compatible API
                             ▼
                    ┌─────────────────┐
                    │    LiteLLM      │
                    │   Gateway       │
                    │     :4000       │
                    └────────┬────────┘
                             │
                      model mapping
                             │
                             ▼
                    ┌─────────────────┐
                    │      Groq       │
                    │ GPT-OSS-120B    │
                    └────────┬────────┘
                             │
                             ▼
                         Response
```

**Gateway runtime foundation: COMPLETE.**

---

## Next Session — Day 4

The next logical step is to move from:

```text
Single Gateway
      ↓
Single Provider
      ↓
Single Model
```

to:

```text
                    ┌─ Provider A
Client → LiteLLM ───┼─ Provider B
                    └─ Fallback / Retry
```

Focus:

**Provider abstraction → fallback → routing → failure handling.**

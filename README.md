# vLLM Stack Setup

This stack runs an on-prem LiteLLM gateway in front of two vLLM model servers:

- `coding-chat` routes to `Qwen/Qwen3-Coder-30B-A3B-Instruct-FP8`
- `coding-autocomplete` routes to `Qwen/Qwen2.5-Coder-7B`

The Compose file also includes a one-time Hugging Face preload job so the initial model downloads happen explicitly before the serving containers start.

## Directory Structure

```text
.
├── docker-compose.yml
├── litellm_config.yaml
├── Caddyfile
└── .env.template
```

## Required `.env` Values

Create `.env` from your template and set the required secrets:

```bash
cp .env.template .env
nano .env
```

Minimum required values:

```bash
HF_TOKEN=hf_your_hugging_face_token
LITELLM_MASTER_KEY=sk-your-litellm-master-key
POSTGRES_PASSWORD=replace-with-a-strong-password
```

One way generate strong random secrets is using OpenSSL:

```bash
echo “LITELLM_MASTER_KEY=sk-$(openssl rand -hex 32)” >> /dev/null  # preview
echo “POSTGRES_PASSWORD=$(openssl rand -hex 32)”     >> /dev/null  # preview
```

Note that the Hugging Face token must have access to the Qwen model repositories being pulled.



## Pull Images

Pull the container images first:

```bash
docker compose pull
```

## Preload the Hugging Face Models

Before starting the runtime stack, preload the model files into the shared `huggingface_cache` Docker volume:

```bash
docker compose --profile preload run --rm hf-preload
```

This downloads:

```text
Qwen/Qwen3-Coder-30B-A3B-Instruct-FP8
Qwen/Qwen2.5-Coder-7B
```

This step can take a while on first run. To watch the preload process:

```bash
docker compose --profile preload run --rm hf-preload
```

If the preload fails, check that `HF_TOKEN` is present in `.env`, the host has internet access, DNS works from Docker containers, and your Hugging Face account has access to the model repositories.

## Start the Stack

After the preload finishes, start the runtime services:

```bash
docker compose up -d
```

This starts:

```text
vllm-coder
vllm-autocomplete
litellm
postgres
caddy
```

## Watch Startup Logs

Watch the model servers load:

```bash
docker compose logs -f vllm-coder
```

In a second terminal, watch the autocomplete model:

```bash
docker compose logs -f vllm-autocomplete
```

Watch LiteLLM:

```bash
docker compose logs -f litellm
```

## Health Checks

Check running containers:

```bash
docker compose ps
```

Check LiteLLM locally:

```bash
curl http://localhost:4000/health
```

List models exposed through LiteLLM:

```bash
curl http://localhost:4000/v1/models \
  -H "Authorization: Bearer $LITELLM_MASTER_KEY"
```

Expected logical model names:

```text
coding-chat
coding-best
coding-autocomplete
```

## LiteLLM Model Routing

The intended `litellm_config.yaml` mapping is:

```yaml
model_list:
  - model_name: coding-chat
    litellm_params:
      model: openai/qwen3-coder-30b-a3b
      api_base: http://vllm-coder:8000/v1
      api_key: EMPTY

  - model_name: coding-best
    litellm_params:
      model: openai/qwen3-coder-30b-a3b
      api_base: http://vllm-coder:8000/v1
      api_key: EMPTY

  - model_name: coding-autocomplete
    litellm_params:
      model: openai/qwen2.5-coder-7b
      api_base: http://vllm-autocomplete:8000/v1
      api_key: EMPTY

general_settings:
  master_key: os.environ/LITELLM_MASTER_KEY
  database_url: os.environ/DATABASE_URL

litellm_settings:
  request_timeout: 600
  num_retries: 2
  drop_params: true
  set_verbose: false
```

`coding-chat` and `coding-best` are separate logical routes that point to the same physical vLLM backend. This avoids trying to run `Qwen3-Coder-Next-FP8`, `Qwen3-Coder-30B-A3B-Instruct-FP8`, and an autocomplete model concurrently on a single DGX Spark memory pool.

## Create Per-User API Keys

Once LiteLLM is running, create a key for each user via the LiteLLM API:

```bash
curl -X POST http://localhost:4000/key/generate \
  -H "Authorization: Bearer $LITELLM_MASTER_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "user_id": "alice",
    "key_alias": "alice-dev",
    "max_budget": 1000,
    "budget_duration": "30d"
  }'
```

Hand each user their returned `key` value. They use it as a Bearer token in Continue, Pi Agent, or another OpenAI-compatible client.

## Client Configuration

Point clients at LiteLLM:

- **API Base:** `http://<server-ip>/v1`
- **API Key:** the per-user key generated above
- **Chat Model:** `coding-chat`
- **Autocomplete Model:** `coding-autocomplete`

For Caddy-backed HTTPS, use:

- **API Base:** `https://<your-llm-hostname>/v1`

## Example API Test

Test chat completions through LiteLLM:

```bash
curl -X POST http://localhost:4000/v1/chat/completions \
  -H "Authorization: Bearer $LITELLM_MASTER_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "coding-chat",
    "messages": [
      {"role": "user", "content": "Write a Python function that validates an IPv4 CIDR string."}
    ],
    "temperature": 0.2,
    "max_tokens": 512
  }'
```

Test the autocomplete route:

```bash
curl -X POST http://localhost:4000/v1/completions \
  -H "Authorization: Bearer $LITELLM_MASTER_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "coding-autocomplete",
    "prompt": "def is_valid_cidr(value):\n    ",
    "temperature": 0.1,
    "max_tokens": 128
  }'
```

## Access UIs

| Service     | URL                        |
|-------------|----------------------------|
| LiteLLM UI  | `http://localhost:4000/ui` |
| LiteLLM API | `http://localhost:4000/v1` |

With Caddy and DNS configured, use the HTTPS hostname from your `Caddyfile` instead of `localhost`.

## Common Operations

Stop the stack:

```bash
docker compose down
```

Restart only LiteLLM after editing `litellm_config.yaml`:

```bash
docker compose restart litellm
```

Restart a model server:

```bash
docker compose restart vllm-coder
```

Update images:

```bash
docker compose pull
docker compose up -d
```

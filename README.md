# vLLM Stack Setup

## Directory Structure

```
.
├── docker-compose.yml
├── litellm_config.yaml
└── .env
```

## Start the Stack

```bash
# 1. Fill in your .env values
cp .env.template .env && nano .env

# 2. Pull images and start
docker compose up -d

# 3. Watch vLLM load the model (takes a few minutes first run)
docker compose logs -f vllm
```

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

Hand each user their `key` value. They use it as a Bearer token in continue.dev / Pi Agent.

## Client Configuration (continue.dev / Pi Agent)

Point clients at:

- **API Base:** `http://<server-ip>:4000/v1`
- **API Key:** the per-user key generated above
- **Model:** `qwen2.5-coder-72b`

## Access UIs

|Service   |URL                     |
|----------|------------------------|
|LiteLLM UI|http://localhost:4000/ui|
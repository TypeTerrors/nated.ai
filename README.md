

# DGX Spark LLM Stack – Ollama + AnythingLLM + Cloudflare Tunnel (`nated.ai`)

This project runs **local LLMs on your DGX Spark** and exposes a nice web UI over the internet via **Cloudflare Tunnel** at **`https://nated.ai`**.

It includes:

- **Ollama** – runs local language models with GPU acceleration on port **11434**.
- **AnythingLLM** – a full RAG/chat UI that talks to Ollama (and optionally OpenAI) on port **3001**.
- **Cloudflare Tunnel (cloudflared)** – securely publishes AnythingLLM at `nated.ai` using your Cloudflare account.

---

## 1. Stack Overview

### Services

- **ollama**
  - Docker image: `ollama/ollama:latest`
  - Host port: `11434`
  - Data volume: `./ollama_data:/root/.ollama`
  - Uses the NVIDIA runtime with `NVIDIA_VISIBLE_DEVICES=all`.

- **anythingllm**
  - Docker image: `mintplexlabs/anythingllm:latest`
  - Host port: `3001`
  - Data volume: `./anythingllm_storage:/app/server/storage`
  - Uses **Ollama** as the default LLM and embedding engine.
  - Can optionally use **OpenAI** via your `OPENAI_API_KEY`.

- **tunnel-spark**
  - Docker image: `cloudflare/cloudflared:latest`
  - Runs `cloudflared tunnel run` with your `TUNNEL_TOKEN`.
  - Exposes AnythingLLM at `https://nated.ai` through Cloudflare’s network.

---

## 2. Environment Variables (`.env`)

In the same folder as `docker-compose.yml`, create a `.env` file:

```env
OPENAI_API_KEY=sk-xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
TUNNEL_TOKEN=eyJhIjoi...paste-from-cloudflare...
```

- `OPENAI_API_KEY` – your real OpenAI API key.
- `TUNNEL_TOKEN` – token from Cloudflare when you create the tunnel for `nated.ai`.

Docker Compose will automatically substitute `${OPENAI_API_KEY}` and `${TUNNEL_TOKEN}` in `docker-compose.yml`.

---

## 3. Starting the Stack

From the DGX Spark, in the directory with `docker-compose.yml`:

```bash
docker compose pull
docker compose up -d
```

Check containers:

```bash
docker ps
```

You should see `ollama`, `anythingllm`, and `tunnel-spark` running.

---

## 4. Accessing AnythingLLM Locally (LAN)

1. On the DGX, get its IP address:

   ```bash
   hostname -I
   ```

2. From your Mac (or any machine on the same network), open in a browser:

   ```text
   http://&lt;DGX_IP&gt;:3001
   # example: http://192.168.1.50:3001
   ```

3. First-time AnythingLLM setup:

   - Create an admin account.
   - Open **Settings → LLM Preference**.
   - Choose **Ollama** as the LLM provider.
   - Confirm:
     - Base URL: `http://ollama:11434`
     - Default model: `llama3.1:8b-instruct-q4_K_M` (or any other installed model).
   - Save.

You now have a local, GPU-accelerated assistant powered by Ollama, managed through AnythingLLM.

---

## 5. Publishing AnythingLLM at `https://nated.ai` (Cloudflare Tunnel)

### 5.1. Create a Cloudflare Tunnel & Token

1. Log into the **Cloudflare dashboard** for `nated.ai`.
2. Go to **Zero Trust → Networks → Tunnels**.
3. Create a new tunnel (for example, name it `spark-tunnel`).
4. Choose the **Cloudflared** connector option.
5. Cloudflare will show a command containing a `--token` value, e.g.:

   ```bash
   cloudflared tunnel run --token &lt;TUNNEL_TOKEN&gt;
   ```

6. Copy the token portion (`&lt;TUNNEL_TOKEN&gt;`) into your `.env` file as `TUNNEL_TOKEN`.

7. Restart the tunnel container:

   ```bash
   docker compose up -d tunnel-spark
   ```

The `tunnel-spark` service now connects your DGX to Cloudflare.

### 5.2. Map `nated.ai` to AnythingLLM

In the Cloudflare dashboard:

1. Edit your tunnel configuration.
2. Add a **Public Hostname**:
   - Hostname: `nated.ai`
   - Type: HTTP
   - URL / Service: `http://&lt;DGX_LOCAL_IP&gt;:3001`  
     (Use the same IP from `hostname -I`.)

3. Make sure `nated.ai`’s DNS record is proxied by Cloudflare (orange cloud icon).

Once the tunnel is healthy, you can reach AnythingLLM from anywhere via:

```text
https://nated.ai
```

---

## 6. Managing Local Models with Ollama

Ollama runs as a server inside the `ollama` container. You add or manage models using the `ollama` CLI **inside that container**.

### 6.1. Open a Shell in the Ollama Container

On the DGX:

```bash
docker exec -it ollama bash
```

Now you are inside the container.

### 6.2. List Installed Models

```bash
ollama list
```

This shows all models currently downloaded and ready to use.

### 6.3. Download (Pull) a New Model

Use `ollama pull` with the model name you want. Examples:

```bash
# Smaller / faster models
ollama pull llama3.1:8b-instruct-q4_K_M
ollama pull mistral:7b-instruct

# Larger, more capable models (if you have the memory and patience)
ollama pull llama3.1:70b
ollama pull gemma2:27b
```

Ollama will download the model and store it in `/root/.ollama` (which is mapped to `./ollama_data` on the host).

### 6.4. Test a Model in the Container

```bash
ollama run llama3.1:8b-instruct-q4_K_M
```

Type a quick prompt to verify it works, then press `Ctrl+C` to exit.

---

## 7. Using New Models in AnythingLLM

Once Ollama has a model, AnythingLLM can use it by referring to the model name.

You have **two ways** to set which model AnythingLLM uses:

### Option A – Change the Default Model in `docker-compose.yml`

In `docker-compose.yml`, update:

```yaml
environment:
  - OLLAMA_MODEL_PREF=llama3.1:8b-instruct-q4_K_M
```

to the new model name, e.g.:

```yaml
  - OLLAMA_MODEL_PREF=llama3.1:70b
```

Then restart AnythingLLM:

```bash
docker compose up -d anythingllm
```

AnythingLLM will now default to the new model for chats (assuming it exists in Ollama).

### Option B – Change the Model in the AnythingLLM UI

1. Go to the AnythingLLM web UI (`http://&lt;DGX_IP&gt;:3001` or `https://nated.ai`).
2. Open **Settings → LLM Preference**.
3. Select **Ollama** as provider if it isn’t already.
4. Set the **Model** field to the new model name, e.g.:

   - `llama3.1:8b-instruct-q4_K_M`
   - `llama3.1:70b`
   - `mistral:7b-instruct`
   - etc.

5. Save.

From now on, that workspace will send prompts to the selected Ollama model.

---

## 8. Using OpenAI and Local Models Together

Because `OPENAI_API_KEY` is passed into the AnythingLLM container, you can configure OpenAI as an additional provider:

1. In the AnythingLLM UI, open **Settings → LLM Preference**.
2. Choose **OpenAI** (or the equivalent option).
3. Set:
   - API key: it should read from the environment, or you can paste it manually.
   - Model: e.g. `gpt-4.1`, `gpt-4.1-mini`, etc.
4. Save.

Now you can:

- Use **Ollama** for most local/private/general work.
- Switch specific workspaces or agents to **OpenAI** for more powerful or specialized tasks.

---

## 9. Common Docker Commands

From the DGX project directory:

```bash
# Start or update the stack
docker compose pull
docker compose up -d

# View logs
docker logs -f ollama
docker logs -f anythingllm
docker logs -f tunnel-spark

# Shell into Ollama container
docker exec -it ollama bash

# Stop everything
docker compose down
```

# Qwen2.5-Coder on RHEL 9.7 (AMD EPYC 9825, CPU-only) — Opinionated Deployment Runbook

**Target host:** `sj-sv2-jenkins-83` · **OS:** RHEL 9.7 (Plow)
**CPU:** 2 × AMD EPYC 9825 (Zen 5 "Turin", 144c/socket, 576 threads, AVX-512) · **RAM:** 1 TiB · **GPU:** none → **CPU-only inference**
**Backend:** Ollama (0.24.x) · **UI:** Open WebUI (0.9.x) · **Proxy:** Nginx, plain HTTP on the LAN IP (no SSL)

--- Neha

## The decision, up front

This is how I'd actually run it for a dev team, and the whole runbook is built around it:

- **Default model = `qwen-coder-32k`** — Qwen2.5-Coder **14B at a 32K context**. This is the daily driver. On CPU it's the best quality-for-speed point and 32K is enough context for almost all real coding work.
- **`qwen-coder-32b` = the heavy hitter** — pulled out only for hard refactors / deep reasoning where you'll accept slower tokens for better answers.
- **`qwen-coder-128k` = special-occasion only** — 128K context exists for genuine whole-repo ingestion. It's slower per token and lower quality than native context, so it is *not* the everyday model.

You have 1 TB RAM, so all three stay resident — but the team defaults to the fast one and reaches for the others deliberately.

Two non-negotiables this hardware forces:
1. **No GPU → memory-bandwidth-bound.** Don't use all 576 threads; pin to one socket and use ~64 threads. NUMA locality matters more than core count.
2. **Root is only 70 GB.** Models go on `/mnt/raid0` (7 TB), never on `/`.

And the two things you explicitly asked for are handled end-to-end: **token-by-token streaming that stays fast** (Section 7) and **Nginx on the local IP with no SSL** (Section 6).

---

## 1. Prep the system

```bash
dnf -y update
dnf -y install numactl numactl-libs htop tmux jq curl tar policycoreutils-python-utils

# Confirm AVX-512 (expect avx512f, avx512_bf16, ...)
grep -o 'avx512[a-z_0-9]*' /proc/cpuinfo | sort -u
# Confirm 2 NUMA nodes: node0 = 0-143,288-431  | node1 = 144-287,432-575
numactl --hardware
```

## 2. Put model storage on the RAID array (not on root)

```bash
mkdir -p /mnt/raid0/ollama/models
```

## 3. Install Ollama + configure it for this box

```bash
curl -fsSL https://ollama.com/install.sh | sh
ollama --version
```

Drop-in config — storage on RAID, listen on the LAN, models stay hot, long-context KV savings, modest CPU-appropriate concurrency:

```bash
mkdir -p /etc/systemd/system/ollama.service.d

cat > /etc/systemd/system/ollama.service.d/override.conf << 'EOF'
[Service]
Environment="OLLAMA_MODELS=/mnt/raid0/ollama/models"
Environment="OLLAMA_HOST=0.0.0.0:11434"
Environment="OLLAMA_KEEP_ALIVE=-1"
Environment="OLLAMA_FLASH_ATTENTION=1"
Environment="OLLAMA_KV_CACHE_TYPE=q8_0"
Environment="OLLAMA_NUM_PARALLEL=2"
Environment="OLLAMA_MAX_LOADED_MODELS=2"
Environment="OLLAMA_NUM_THREADS=64"
EOF

chown -R ollama:ollama /mnt/raid0/ollama
systemctl daemon-reload
systemctl restart ollama
```

**NUMA pinning** — the biggest CPU lever. Pin the engine to one socket so compute and memory stay local:

```bash
which ollama   # confirm path, usually /usr/local/bin/ollama

cat > /etc/systemd/system/ollama.service.d/numa.conf << 'EOF'
[Service]
ExecStart=
ExecStart=/usr/bin/numactl --cpunodebind=0 --membind=0 /usr/local/bin/ollama serve
EOF

systemctl daemon-reload
systemctl restart ollama
```

## 4. Pull the models

```bash
ollama pull qwen2.5-coder:14b     # base for the default profile
ollama pull qwen2.5-coder:32b     # base for the heavy + 128k profiles

ollama list
du -sh /mnt/raid0/ollama/models
df -h /            # root should be basically untouched
```

## 5. Build the three profiles (the default first)

**`qwen-coder-32k` — the default everyone uses (14B, 32K native context):**

```bash
cat > /tmp/Modelfile.default << 'EOF'
FROM qwen2.5-coder:14b
PARAMETER num_ctx 32768
PARAMETER num_batch 512
PARAMETER num_thread 64
PARAMETER temperature 0.2
PARAMETER top_p 0.9
EOF
ollama create qwen-coder-32k -f /tmp/Modelfile.default
```

**`qwen-coder-32b` — the heavy hitter, for hard problems:**

```bash
cat > /tmp/Modelfile.heavy << 'EOF'
FROM qwen2.5-coder:32b
PARAMETER num_ctx 32768
PARAMETER num_batch 512
PARAMETER num_thread 96
PARAMETER temperature 0.2
PARAMETER top_p 0.9
EOF
ollama create qwen-coder-32b -f /tmp/Modelfile.heavy
```

**`qwen-coder-128k` — special-occasion whole-repo context (32B + YaRN):**

```bash
cat > /tmp/Modelfile.bigctx << 'EOF'
FROM qwen2.5-coder:32b
PARAMETER num_ctx 131072
PARAMETER num_batch 256
PARAMETER num_thread 96
PARAMETER temperature 0.2
PARAMETER top_p 0.9
PARAMETER rope_scaling_type yarn
PARAMETER rope_scaling_factor 4.0
PARAMETER rope_scaling_orig_ctx 32768
EOF
ollama create qwen-coder-128k -f /tmp/Modelfile.bigctx

ollama list   # qwen-coder-32k, qwen-coder-32b, qwen-coder-128k
```

> Reminder on the 128K profile: KV cache gets large and tokens/sec drops. It's the deliberate exception, not the default — which is exactly why the everyday model is `qwen-coder-32k`.

## 6. Install Open WebUI + Nginx (local IP, plain HTTP, no SSL)

### Docker + Open WebUI

```bash
dnf -y install dnf-plugins-core
dnf config-manager --add-repo https://download.docker.com/linux/rhel/docker-ce.repo
dnf -y install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
systemctl enable --now docker

docker run -d \
  --name open-webui \
  --restart unless-stopped \
  -p 127.0.0.1:3000:8080 \
  --add-host=host.docker.internal:host-gateway \
  -e OLLAMA_BASE_URL=http://host.docker.internal:11434 \
  -v open-webui:/app/backend/data \
  ghcr.io/open-webui/open-webui:main
```

> Note: Open WebUI is bound to `127.0.0.1:3000` on purpose — Nginx is the only thing the network sees, on port 80. Users go to `http://<server-ip>/`.

### Nginx — reverse proxy on the LAN IP, HTTP only, tuned for streaming

```bash
dnf -y install nginx

cat > /etc/nginx/conf.d/openwebui.conf << 'EOF'
server {
    listen 80;
    server_name _;            # responds on the server's IP, no domain/SSL needed

    # Big code prompts/responses
    client_max_body_size 100m;

    location / {
        proxy_pass http://127.0.0.1:3000;

        # --- WebSocket / upgrade support (Open WebUI uses it) ---
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # --- STREAMING: must be off or tokens arrive in one lump ---
        proxy_buffering off;
        proxy_cache off;
        proxy_request_buffering off;
        chunked_transfer_encoding on;

        # --- Long generations must not time out mid-stream ---
        proxy_read_timeout 600s;
        proxy_send_timeout 600s;
    }
}
EOF

nginx -t
systemctl enable --now nginx

# SELinux: allow nginx to make the proxy connection
setsebool -P httpd_can_network_connect 1
```

### Firewall — open only port 80

```bash
firewall-cmd --permanent --add-service=http     # port 80
firewall-cmd --reload
```

Now browse to **`http://<server-ip>/`**. First account created = admin.

## 7. Make streaming appear fast (and stay fast)

Streaming has two halves: tokens must *leave* Ollama incrementally, and nothing in the path may *buffer* them. Both are handled:

- **Ollama** streams token-by-token by default; **Open WebUI** renders progressively as they arrive. No flag needed.
- **The usual streaming killer is a buffering proxy** — already neutralized in Section 6 with `proxy_buffering off`, `proxy_request_buffering off`, `proxy_cache off`, and `chunked_transfer_encoding on`. This is the single most common reason "responses appear all at once," so don't omit those lines.
- **Keep first-token latency low:** `OLLAMA_KEEP_ALIVE=-1` (Section 3) keeps the model resident so there's no reload pause before the first token.
- **Verify the stream is live** (you should see text dribble out, not dump at the end):

```bash
curl -N http://localhost:11434/api/generate \
  -d '{"model":"qwen-coder-32k","prompt":"Explain Python decorators in 3 short paragraphs."}'
```

`-N` disables curl's own buffering; watch the tokens flow.

In Open WebUI, set **Settings → Admin → Models → default model = `qwen-coder-32k`** so every new chat starts on the fast profile; the heavy and 128K profiles are one dropdown click away when someone needs them.

## 8. Tune the thread count (do this once)

Best `num_thread` is empirical on a bandwidth-bound box — it plateaus then drops, so find the peak:

```bash
PROMPT="Refactor: def f(x):return [i for i in range(x) if all(i%j for j in range(2,i))]"
for T in 16 32 48 64 96 144; do
  echo "===== threads=$T ====="
  numactl --cpunodebind=0 --membind=0 env OLLAMA_NUM_THREADS=$T \
    ollama run qwen-coder-32k --verbose "$PROMPT" 2>&1 | grep -i "eval rate"
done
```

Put the winner into `override.conf` (`OLLAMA_NUM_THREADS`) and the Modelfiles (`num_thread`), then `systemctl restart ollama` and `ollama create` the profiles again.

## 9. Verify

```bash
systemctl status ollama  --no-pager | head -3
systemctl show  ollama   -p ExecStart        # should show numactl wrapping
du -sh /mnt/raid0/ollama/models; df -h / | tail -1
curl -s http://localhost:11434/api/tags | jq '.models[].name'
docker ps --filter name=open-webui
curl -s -o /dev/null -w "%{http_code}\n" http://localhost/   # 200 via nginx
```

## 10. Troubleshooting

- **Responses appear all at once:** a buffering line is missing in the Nginx block — re-check `proxy_buffering off` and friends; reload nginx.
- **502/“can’t reach Ollama” from the UI:** confirm `OLLAMA_HOST=0.0.0.0:11434`, the `--add-host` flag, and `setsebool -P httpd_can_network_connect 1`.
- **`/` fills up:** `OLLAMA_MODELS` wasn’t applied — stop Ollama, move the dir to `/mnt/raid0`, set the env, restart.
- **Slower with more threads:** expected — back off; you saturated memory bandwidth.
- **128K crawls / OOM-ish:** confirm flash attention + `q8_0` KV cache; otherwise use `qwen-coder-32k`.

---

### Bottom line
Day to day, everyone is on **14B @ 32K (`qwen-coder-32k`)** behind a plain-HTTP Nginx at `http://<server-ip>/`, with **streaming on and unbuffered** so tokens appear immediately and keep flowing. The 32B and 128K profiles are there in the dropdown for the moments that actually need them.

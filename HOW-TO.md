# HOW-TO: single 3090, managed with Dockhand

Run this repo's compose stack on **one RTX 3090**, with [Dockhand](https://dockhand.pro/)
as the compose manager instead of the `docker compose` CLI. Everything else
(image, model prep, start scripts) works exactly as in
[docs/docker.md](docs/docker.md) — this only changes how you deploy and
watch the stack.

The compose file already targets a single card on this host: it reserves
`device_ids: ["1"]` (the first 3090; GPU 0 is the RTX PRO 6000) and runs
`TP=1`. If your 3090 sits at a different index, edit `device_ids:` in
`docker-compose.yml` first.

## 1. Host prerequisites

Same as the plain-docker path (docs/docker.md): NVIDIA driver that speaks
CUDA 13 (≥ 580), Docker Engine, and the
[NVIDIA Container Toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html)
installed. The 250 W power limit, if you want it, is a host setting
(`sudo nvidia-smi -pl 250` on the 3090) — neither the container nor Dockhand
can set it.

## 2. Install Dockhand

```bash
docker run -d --name dockhand --restart unless-stopped \
  -p 3000:3000 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /opt/dockhand:/app/data \
  fnsys/dockhand:latest
```

- **Use the matching host path (`/opt/dockhand:/app/data`), not a named
  volume.** This compose file uses the relative volume path
  `${MODELS_DIR:-./models}:/app/models`. The *host* Docker daemon resolves
  that path from the stack's compose directory, which lives under
  Dockhand's data dir — so the path must exist at the same location on the
  host (Dockhand docs: "matching paths"). `/opt/dockhand/stacks/...` inside
  the container = `/opt/dockhand/stacks/...` on the host, and `./models`
  works with no translation.
- First launch has **authentication disabled**: open
  [http://localhost:3000](http://localhost:3000) and go to
  **Settings → Authentication** to create an admin user before pointing it
  at a real GPU box.

## 3. Add the qwen stack

**Option A — Git stack (recommended).** Point Dockhand at the repo:
**Compose → Create stack → Git**, repository `https://github.com/<you>/qwen38-27b-rtx3090`
(your fork — the single-GPU compose must be pushed there first), compose file
`docker-compose.yml`, stack name e.g. `qwen`. Dockhand clones the whole repo
directory, so `env_file: .env` and `./models` both resolve inside the clone,
and the stack **auto-syncs on new commits** (optionally via a GitHub
webhook) — so a later upstream sync you push redeploy the stack for you.

**Option B — Internal stack.** **Compose → Create stack**, name it, paste the
full contents of `docker-compose.yml` from your checkout. Dockhand stores it
in its own stacks dir (fine with the matching-path setup from §2); the env
variables you add in the UI are written to the stack's `.env`, which the
compose file reads via `env_file:`.

## 4. Environment variables

In the stack's environment panel (or the repo's `.env` for a Git stack):

```ini
COMPOSE_PROFILES=single      # REQUIRED — see note below
VLLM_API_KEY=<openssl rand -hex 24>

# optional knobs, all from docs/docker.md:
# CTX=long  KV=kvarn  MAX_LEN=  MAX_SEQS=  EXTRA_ARGS=...
# FAST single-user decode (drafter is fetched by `prepare`):
# SPEC=dflash2  DFLASH_TOKENS=15  PREFIX_CACHE=1
```

- **`COMPOSE_PROFILES` is the one non-obvious variable.** `single` and
  `batch` are compose profiles and `prepare` is the only unprofiled service:
  a deploy *without* a profile runs `prepare` (model download + requantize,
  idempotent) and then starts nothing. `COMPOSE_PROFILES=single` is a
  built-in variable Dockhand's editor recognizes and passes to compose at
  deploy time.
- Keep `VLLM_API_KEY` a **regular** variable, not a Dockhand secret. Secrets
  are injected only when the compose file references them in
  `environment:`; this stack passes its `.env` whole via `env_file:`, so a
  "secret" value would reach the container as an empty string.
- On WSL2 hosts add `GPU_UTIL=0.93` and (with `SPEC=dflash2`)
  `VLLM_WSL2_ENABLE_PIN_MEMORY=1` — see the WSL2 section of docs/docker.md.

## 5. Deploy

Click **Deploy** (or tick "Deploy immediately" on create). What happens, in
order — watch it in the UI's container logs:

1. Image pull: `ghcr.io/syv-ai/qwen38-27b-rtx3090:latest` (~9.5 GB, prebuilt
   per upstream commit by CI; `pull_policy: missing` so it is reused after).
2. `prepare` service: model download into `./models` (~20 GB) + requantize +
   DFlash2 drafter fetch. CPU-only, idempotent.
3. `single` server starts; first boot does torch.compile / CUDA graphs /
   FlashInfer JIT (~2–3 min). The compose healthcheck probes
   `http://127.0.0.1:18020/health` with a 900 s start period — expect the
   tile to show unhealthy for up to ~15 min on the very first boot only;
   later boots are ~1 min (the compile cache lives in the `qwen-cache`
   volume).

Verified live: `curl http://localhost:18020/health`, then a chat request with
`VLLM_API_KEY`. Expected on one 3090: single ~112–115 tok/s decode, batch
~950–1,040 tok/s (measured, docs/docker.md).

## 6. Day-to-day

- **Logs / restart / stop / view** — the stack row on Dockhand's Compose
  stacks page; per-container logs under the expanded tiles.
- **single ↔ batch** — set `COMPOSE_PROFILES=batch` (or `single`) in the env
  panel and **redeploy**. Only one mode at a time: the card is reserved
  either way.
- **Model location** — `./models` lands in the stack's compose directory
  (inside `/opt/dockhand/...`). To share one download with a venv install,
  set `MODELS_DIR=/abs/path/to/models` in the env panel and redeploy.
- **Upgrades** — with the Git stack, a pushed commit touching
  `docker-compose.yml` (or its `.env`) triggers a redeploy; you can also hit
  the **Deploy** button manually. Pin a known build by editing the
  `image:` line to `ghcr.io/syv-ai/qwen38-27b-rtx3090:sha-<7>`.
- **Tearing it down** — stack **Stop** (keeps `./models` + the
  `qwen-cache` volume); **Delete** stack plus `docker volume rm
  <project>_qwen-cache` if you want it gone entirely.

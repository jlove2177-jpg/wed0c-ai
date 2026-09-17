# WeD0c AI — Standard Operating Procedure

## Project Map

```
~/Projects/
├── murasaki/          # 3D geisha waifu — morph targets, lip-sync, 3D viewer
├── meme-factory/      # Trend-to-meme pipeline, contracts, scripts
├── branded-dash/      # White-label customer dashboard generator
├── fable-pipeline/    # x402 gateway + runner + improve server
├── akira-os/          # Original 2D dashboard (legacy)
├── wed0c-ai/          # Business proposals, freight sales, sales enablement
├── OpenMontage/       # YouTube Shorts pipeline (4 channels)
├── fable-402/         # x402 gateway codebase
└── voicebox/          # TTS server
```

---

## Service Status

### Active Ports
| Port | Service | Type |
|------|---------|------|
| 8080 | Ornith (llama-server) | systemd |
| 17493 | Kokoro TTS | systemd |
| 8188 | ComfyUI | systemd |
| 5678 | n8n | systemd |
| 4020 | Fable Gateway | systemd |
| 9403 | Murasaki UI / Akira UI | background process |
| 9404 | Fable Runner | background process |
| 9405 | Branded Dashboard | background process |
| 17494 | Coqui TTS | background process |

### Health Check
```bash
hermes cron list                                          # verify cron jobs
for svc in llama-server kokoro comfyui n8n fable-gateway ornith-health; do
  systemctl --user is-active "$svc"
done
curl -s http://127.0.0.1:9403/dashboard/viewer.html       # Murasaki
curl -s http://127.0.0.1:8080/v1/models                   # Ornith
curl -s http://127.0.0.1:9404/                             # Fable Runner
```

---

## Service Management

### Start Murasaki
```bash
cd ~/Projects/murasaki && python3 -m http.server 9403
```
URL: http://127.0.0.1:9403/dashboard/viewer.html
Tailscale: http://100.87.57.33:9403/dashboard/viewer.html

### Start Fable Runner
```bash
python3 ~/Projects/fable-pipeline/runner/fable-runner.py
```
URL: http://127.0.0.1:9404

### Restart Ornith (after OOM)
```bash
systemctl --user restart comfyui.service    # clear VRAM
systemctl --user restart llama-server.service
```
Wait 30s, verify with curl :8080.

### GPU Sequencing (free VRAM for ComfyUI)
```bash
systemctl --user stop llama-server.service   # frees ~9 GB
# ... run ComfyUI generation ...
systemctl --user restart llama-server.service
```

---

## VRAM Budget (24 GB RX 7900 XTX)

| Configuration | Ornith | Free |
|--------------|--------|------|
| 35B Q6_K | ~22 GB | ~1 GB |
| **9B Q4_K_M** (current) | **~9 GB** | **~15 GB** |

Current 9B model: `~/Downloads/ornith.gguf` (5.6 GB file, 9.3 GB VRAM)
Old 35B Q6_K: `~/Downloads/ornith-35b-Q6_K.gguf` (21.8 GB file)

---

## MOA Configuration

| Role | Model | Provider |
|------|-------|----------|
| Aggregator | Ornith 1.0 | custom (local) |
| Reference | Nemotron 3 Ultra | openrouter |
| Reference | Nemotron 3 Super 120B | openrouter (free) |
| Reference | Hermes 3 405B | openrouter (free) |
| Reference | Llama 3.3 70B | openrouter (free) |
| Reference | Gemma 4 31B | openrouter (free) |
| Reference | DeepSeek Chat | deepseek |
| Reference | DeepSeek V4 Flash | deepseek |
| Reference | Gemini 2.5 Flash | google |
| Reference | Ornith 1.0 | custom (local) |

---

## Murasaki 3D

### Viewer Controls
- **Drag** to orbit
- **Click vowels** (A/E/I/O/U) — morph target lip-sync
- **Click emotions** (smile/sad/wow/blink) — facial expressions
- **Click hello** — cycles through mouth shapes
- **Auto-blink** — every 3 seconds

### Files
| File | Purpose |
|------|---------|
| `dashboard/viewer.html` | Three.js 3D viewer with morph targets |
| `frames/murasaki.glb` | 3D model with shape keys + animation |
| `murasaki.blend` | Blender source file |
| `frames/waifu_reference.mp4` | AI-generated video reference |
| `frames/geisha_reference.jpg` | Geisha art reference |

### Export from Blender
1. Open `murasaki.blend` in Blender GUI
2. Make edits (sculpt, rig, texture)
3. File → Export → glTF 2.0 (.glb)
4. Save as `~/Projects/murasaki/frames/murasaki.glb`
5. Refresh viewer — model updates automatically

---

## Cron Jobs

| Job | Schedule | Status |
|-----|----------|--------|
| llama-watchdog | every 1m | ok |
| kanban-dispatch-loop | every 2m | paused |
| fable-health | every 5m | ok |
| hardware-local-ai-scrape | every 12h | ok |
| hardware-to-home-shorts | every 12h | ok |
| agentic-eye-shorts | every 12h | ok |
| serendipity-shorts | every 12h | ok |
| laugh-track-shorts | every 12h | ok |

Next scrape batch: ~01:20 AM daily.

---

## Rate Limits (2026-07-05)

| Provider | Status | Fallback |
|----------|--------|----------|
| Grok / xAI | ❌ Exhausted | Removed from curator |
| Gemini Omni (image) | ❌ Quota exceeded | ComfyUI |
| FAL.ai | ❌ Balance empty | ComfyUI |
| DeepSeek (API) | ✅ Active | Primary |
| Gemini (text) | ✅ Free tier | Active |
| OpenRouter (free models) | ✅ Active | 5 models in MOA |

---

## Tailscale Network

| Device | IP | Status |
|--------|----|--------|
| Desktop (weeder-ms-7e59) | 100.87.57.33 | ✅ Connected |
| Phone (nosajs-s22-ultra) | 100.65.27.68 | ✅ Connected |
| Cloud VM (voice-agent) | 100.91.237.81 | ✅ Connected |

From phone: open `http://100.87.57.33:9403/dashboard/viewer.html`

---

## Quick Links

| Resource | URL |
|----------|-----|
| Murasaki 3D | http://100.87.57.33:9403/dashboard/viewer.html |
| Ornith API | http://127.0.0.1:8080/v1/models |
| Coqui TTS | http://127.0.0.1:17494/?text=hello |
| Tailscale Admin | https://login.tailscale.com |
| Hugging Face | https://huggingface.co/models?search=ornith |
| DeepSeek Console | https://platform.deepseek.com |
| xAI Console | https://console.x.ai |

---

## Self-Hosted Voice Stack

| Engine | Port | Quality | Status |
|--------|------|---------|--------|
| Kokoro | 17493 | Acceptable | ✅ Active |
| Coqui AI | 17494 | Good | ✅ Active |
| Edge TTS | cloud | Best | ✅ Used for Shorts |

---

*WeD0c AI — SOP v2.0 — Updated 2026-07-06*

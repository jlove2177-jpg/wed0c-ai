# Murasaki Voice Box & Agent OS — Fix Prompt
*Copy this entire file into Claude Code to execute.*

## Mission
Fix the WeD0c AI voice stack and wire agent OS automation on Ubuntu Linux with ROCm GPU.

---

## Step 1: Voice Box — Audit & Fix

### Current State
```
Port 17493 — Kokoro TTS (systemd): may be running, limited voices
Port 17494 — Coqui AI TTS: installed, relies on torch, may have broken after ROCm reinstall
Edge TTS (cloud): working, used for YouTube Shorts narration
```

### Fix Commands
```bash
# Check what's running
systemctl --user status kokoro --no-pager 2>/dev/null | head -10
ss -tlnp | grep -E '17493|17494'

# Test Kokoro
curl -s -o /tmp/kokoro_test.wav "http://127.0.0.1:17493/?text=hello" -m 10 && echo "Kokoro: OK"

# Test Coqui
curl -s -o /tmp/coqui_test.wav "http://127.0.0.1:17494/?text=hello" -m 60 && echo "Coqui: OK"

# Test Edge TTS
edge-tts --text "Hello" --voice en-US-GuyNeural --write-media /tmp/edge_test.wav 2>&1 && echo "Edge: OK"
```

### Fixes
1. **If Kokoro is down:** `systemctl --user restart kokoro`
2. **If Coqui is down:** Start it: `python3 ~/Projects/voicebox/server.py --port 17494 &`  
   If it fails (torch incompatibility from ROCm reinstall): reinstall Coqui with `pip3 install TTS --upgrade`
3. **If Edge TTS is missing:** `pip3 install edge-tts`

### Integration
Edit `~/Projects/OpenMontage/scripts/assemble.py` — the voice section should try Edge TTS first, fall back to Kokoro, then Coqui:
```python
VOICE_PRIORITY = ['edge', 'kokoro', 'coqui']
```

---

## Step 2: Agent OS Automation

### Config Update
Edit `~/.hermes/config.yaml` to set:
```yaml
delegation:
  max_concurrent_children: 3
  max_spawn_depth: 2
  model: /home/weeder/Downloads/ornith.gguf
  provider: custom
  base_url: http://127.0.0.1:8080/v1
```

### SOP Reference
The operating rules are in `~/SOP.md`:
1. **Plan → Present → Approved → Delegate → Review → Loop**
2. Each task: 1 worker + 1 reviewer on different free LLMs + orchestrator reviews
3. Memory and user profile must stay under 50% capacity
4. Always delegate via delegate_task — never do work directly

### Manual Test
```bash
# Verify delegation works
cat ~/.hermes/config.yaml | grep -A5 delegation

# Start the fable pipeline runner
cd ~/Projects/fable-pipeline && python3 runner/fable-runner.py &
curl http://127.0.0.1:9404/

# Start Murasaki viewer
cd ~/Projects/murasaki && python3 -m http.server 9403 &
curl http://127.0.0.1:9403/dashboard/viewer.html
```

---

## Verification
After fixing, run:
```bash
# Voice test
for p in 17493 17494; do
  code=$(curl -s -o /dev/null -w "%{http_code}" http://127.0.0.1:$p/?text=hi -m 5 2>/dev/null)
  echo "Port $p: $code"
done

# Agent OS test
systemctl --user is-active llama-server.service 2>/dev/null
curl -s http://127.0.0.1:8080/v1/models -m 5 | head -c 50

# Dashboard test
curl -s -o /dev/null -w "Murasaki: %{http_code}\n" http://127.0.0.1:9403/dashboard/viewer.html -m 3
```

## Hardware Context
- GPU: AMD Radeon RX 7900 XTX (24 GB VRAM)
- Platform: ROCm 7.1
- Tailscale: desktop 100.87.57.33, phone 100.65.27.68, cloud VM 100.91.237.81
- Services bind to 0.0.0.0 for tailnet access

---

## Step 3: Tailnet Problems — Fix

### Symptom Check
```bash
tailscale status 2>&1  # Shows connected devices
tailscale ip -4        # Shows your tailnet IP
```

### Common Issues
1. **Service binds to localhost** — Murasaki viewer / Fable runner / other services may bind to 127.0.0.1, unreachable from phone or cloud VM.
   **Fix:** Always start with `--bind 0.0.0.0` or set `HOST = "0.0.0.0"` in the server script.
   ```bash
   # Check what a service binds to
   ss -tlnp | grep 9403
   # Should show 0.0.0.0:9403 — if it shows 127.0.0.1:9403, restart with 0.0.0.0
   ```

2. **Phone can't load page** — If Tailscale is on but the page is blank or doesn't load:
   - Try the test page first: `http://[desktop-ip]:9403/dashboard/test.html`
   - Try a different browser on the phone (Chrome, Samsung Internet, Firefox)
   - Toggle Tailscale off/on from the phone app
   - Check if the phone actually shows the desktop as connected in Tailscale app

3. **Cloud VM can't reach services** — The cloud VM (100.91.237.81) should be able to curl all desktop services. Test:
   ```bash
   # From cloud VM via SSH:
   curl -s http://100.87.57.33:8080/v1/models -m 5  # Ornith
   curl -s http://100.87.57.33:9403/dashboard/viewer.html -m 5  # Murasaki
   curl -s http://100.87.57.33:9404/ -m 3  # Fable Runner
   ```

4. **Tailscale Serve not enabled** — For friendly hostnames instead of raw IPs:
   - Visit: `https://login.tailscale.com/f/serve?node=nED9wmvYf111CNTRL`
   - Enable Serve
   - Then: `sudo tailscale serve --https 443 localhost:9403`

### Quick Test
```bash
echo "Testing tailnet connectivity..."
tailscale ping 100.65.27.68 -c 1 2>&1  # Test phone reachability
curl -s -o /dev/null -w "Self: %{http_code}\n" http://100.87.57.33:9403/dashboard/viewer.html -m 3  # Test self via tailnet IP
```

---

## Hermes Agent — Internal SOP (for future use)

### Voice Box
- Kokoro is systemd service (`kokoro.service`), port 17493
- Coqui is manual (`python3 server.py --port 17494`), installed via `pip3 install TTS`
- Edge TTS is `pip3 install edge-tts`, voice `en-US-GuyNeural`
- Priority fallback: Edge → Kokoro → Coqui
- If Coqui fails after torch upgrade: reinstall `pip3 install TTS --upgrade`

### Agent OS
- Always use delegate_task with role='leaf' for work agents
- Worker + reviewer on different free LLMs (Ornith local, OpenRouter free)
- I review the reviewer's output before presenting
- Memory and user profile must stay under 50% at all times
- SOP at ~/SOP.md covers all rules

### Tailnet
- Services MUST bind to 0.0.0.0, not 127.0.0.1, or tailnet devices can't reach them
- Check with `ss -tlnp | grep :PORT`
- Fix with `sed -i 's/127.0.0.1/0.0.0.0/'` on the server script
- Desktop IP: 100.87.57.33 — Phone: 100.65.27.68 — Cloud VM: 100.91.237.81
- Phone loading issue: try test page first, toggle Tailscale, check phone browser cache
- For Claude Code sessions: pass the hardware context (GPU, tailnet IPs, ports) in every delegation

### Blender/3D
- Face retopology: join parts to Sphere, remove_doubles threshold=0.01, then re-gen shape keys
- Shape keys on unified mesh need vertex-position-based offsets (not index-based)
- Export with: `bpy.ops.export_scene.gltf(filepath=OUT, export_format='GLB', use_selection=False, export_materials='EXPORT', export_animations=True)`
- Run with: `blender -b -noaudio -P script.py`
- For long Blender ops: write script to /tmp, then delegate. Expect 30s-5min runtime.

---

## Step 4: SOP Analysis
Read and analyze the full system SOP at `~/SOP.md` before making any changes. This covers:
- All project folders and their purposes
- Active ports and service management
- VRAM budget (Ornith 9B uses ~10 GB, 13 GB free)
- MOA configuration (9 models, Ornith aggregator)
- Cron jobs schedule
- Dev principles (think first, simplicity, surgical changes, goal-driven)
- Quality spectrum and orchestrator loop pattern

After reading SOP.md, report what you found and what you plan to change. Do not modify anything without confirming the SOP's current state first.

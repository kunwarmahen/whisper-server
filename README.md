# whisper-server

OpenAI-compatible speech-to-text server powered by [faster-whisper](https://github.com/SYSTRAN/faster-whisper).

Exposes `POST /v1/audio/transcriptions` — a drop-in replacement for the OpenAI Whisper API. Any app that uses the OpenAI SDK for transcription can point its `base_url` here instead.

## Quick start

```bash
cp .env.example .env        # adjust WHISPER_MODEL if needed
podman compose up -d
```

The model is downloaded on first start and cached in `./models/` — subsequent restarts are instant.

## GPU (NVIDIA)

Uses the NVIDIA Container Toolkit via CDI. One-time host setup (generates the
CDI spec podman reads from `/etc/cdi`):

```bash
sudo nvidia-ctk cdi generate --output=/etc/cdi/nvidia.yaml
nvidia-ctk cdi list          # should list nvidia.com/gpu=all
```

> **podman 4.9.x note:** the toolkit writes a CDI spec at version `0.7.0`, but
> podman 4.9.x only parses up to `0.6.0` (you'll see `unknown field
> "additionalGids"` / `unresolvable CDI devices`). If so, edit
> `/etc/cdi/nvidia.yaml`: set `cdiVersion: 0.6.0` and delete the three
> `additionalGids:` blocks (each is the key plus its two GID lines).

Then start it with **`podman-compose`** (not `podman compose`):

```bash
podman-compose -f docker-compose.yml -f docker-compose.gpu.yml up -d
```

> **Why `podman-compose`?** `podman compose` shells out to the `docker-compose`
> provider, which passes the CDI device name to podman's Docker-compat socket
> where it is treated as a literal file path (`stat nvidia.com/gpu=all: no such
> file or directory`) and the GPU is silently dropped. `podman-compose`
> translates `devices:` into `podman run --device …`, which resolves CDI
> correctly. Install with `pip install podman-compose`.

Verify the GPU is actually in use:

```bash
podman exec whisper-server_whisper_1 nvidia-smi
```

## API

```bash
# Transcribe a file
curl http://localhost:11435/v1/audio/transcriptions \
  -F file=@recording.mp3 \
  -F model=whisper-1

# Health check
curl http://localhost:11435/health
```

## Connecting your app

Set these in your app's `.env`:

```env
STT_PROVIDER=openai
STT_BASE_URL=http://<this-machine-ip>:11435/v1
```

### OpenAI Python SDK

```python
from openai import OpenAI

client = OpenAI(api_key="not-needed", base_url="http://192.168.1.x:11435/v1")

with open("recording.mp3", "rb") as f:
    text = client.audio.transcriptions.create(
        model="whisper-1",
        file=f,
        response_format="text",
    )
print(text)
```

## Model sizes

| Model                             | Size    | Notes                  |
| --------------------------------- | ------- | ---------------------- |
| `Systran/faster-whisper-tiny`     | ~150 MB | Fastest                |
| `Systran/faster-whisper-small`    | ~500 MB |                        |
| `Systran/faster-whisper-medium`   | ~1.5 GB | Default — good balance |
| `Systran/faster-whisper-large-v3` | ~3.1 GB | Most accurate          |

## Port

Default: `11435`. Change the host-side port in `docker-compose.yml` if needed — update `STT_BASE_URL` in your apps accordingly.

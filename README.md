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

Uses direct device passthrough — no CDI setup needed:

```bash
podman compose -f docker-compose.yml -f docker-compose.gpu.yml up -d
```

The GPU compose file maps `/dev/nvidia0`, `/dev/nvidiactl`, `/dev/nvidia-uvm`, and related devices directly into the container. If your system has multiple GPUs, add `/dev/nvidia1` etc. to `docker-compose.gpu.yml`.

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

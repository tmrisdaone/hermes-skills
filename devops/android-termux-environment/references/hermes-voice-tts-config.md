# Hermes Voice & TTS Configuration

Configuration lives in `~/.hermes/config.yaml` under the `voice:` and `tts:` keys.

## TTS Providers

| Provider | API Key Required | Notes |
|----------|-----------------|-------|
| `edge` | No | Free, built-in, good quality. Default. |
| `elevenlabs` | Yes | High quality, needs `ELEVENLABS_API_KEY` |
| `openai` | Yes | Needs `OPENAI_API_KEY` |
| `xai` | Yes | Needs `XAI_API_KEY` |
| `mistral` | Yes | Needs `MISTRAL_API_KEY` |
| `neuphonic` | Optional | Local GGUF model option |
| `piper` | No | Local TTS, runs on-device |

## Key Settings

```yaml
tts:
  provider: edge                          # active TTS provider
  use_gateway: true                       # route TTS through gateway for WebUI / media delivery
  edge:
    voice: en-US-AriaNeural               # voice selection

voice:
  auto_tts: true                          # auto-speak every response
  record_key: ctrl+b                      # key binding to start voice recording
  max_recording_seconds: 120              # max recording length
  beep_enabled: true                      # audible beep on record start/stop
  silence_threshold: 200                  # dB threshold for silence detection
  silence_duration: 3.0                   # seconds of silence before auto-stop

stt:
  enabled: true                           # enable speech-to-text input
  provider: local                         # 'local' = whisper base model (on-device)
  local:
    model: base                           # whisper model size (base/small/medium/large)
```

## Enabling Voice

1. Set `voice.auto_tts: true` for automatic spoken responses
2. Set `tts.use_gateway: true` so audio plays in WebUI/messaging platforms
3. Or test once-off TTS with the `text_to_speech` tool (no config changes needed)
4. Use `stt.enabled: true` + `voice.record_key: ctrl+b` for voice input in CLI

## Voice Selection

Common Edge voices:
- `en-US-AriaNeural` — female, US
- `en-US-JennyNeural` — female, US
- `en-US-GuyNeural` — male, US
- `en-GB-SoniaNeural` — female, UK
- `en-AU-NatashaNeural` — female, AU
- `zh-CN-XiaoxiaoNeural` — female, Mandarin

## Testing

Use the `text_to_speech` agent tool directly to produce audio without changing config:
```
text_to_speech(text="Hello, this is a test.")
```
Returns a file path and media tag for delivery.

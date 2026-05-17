# Voice Bridge Implementation Template
# This template provides a basic FastAPI bridge to decouple Termux:API STT from the Agent engine.

## Bridge Server (voice_bridge.py)
```python
from fastapi import FastAPI, Request
import subprocess
import uvicorn

app = FastAPI()

@app.post("/voice_input")
async def handle_voice(request: Request):
    data = await request.json()
    text = data.get("text", "")
    if not text:
        return {"status": "error", "message": "No text provided"}
    
    # Immediate TTS confirmation
    subprocess.run(["termux-tts-speak", f"You said: {text}"])
    
    # Queue for Agent processing
    with open("voice_command_queue.txt", "a") as f:
        f.write(text + "\\n")

    return {"status": "success", "received": text}

if __name__ == "__main__":
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

## Voice Client (voice_client.py)
```python
import subprocess
import time
import requests

BRIDGE_URL = "http://localhost:8000/voice_input"

def voice_loop():
    while True:
        try:
            result = subprocess.run(["termux-speech-to-text"], capture_output=True, text=True)
            user_text = result.stdout.strip()
            if user_text:
                requests.post(BRIDGE_URL, json={"text": user_text}, timeout=5)
        except Exception as e:
            time.sleep(2)

if __name__ == "__main__":
    voice_loop()
```

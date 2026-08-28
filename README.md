# HandyChat

A Python **Kivy / KivyMD** mobile application that runs HuggingFace causal language models — including Google's **FunctionGemma** — directly on your device.  Users can chat via text or recorded audio, configure every inference hyperparameter, and watch FunctionGemma call built-in tools (calculator, weather, web search, time/date, reminders) in real time.

Inspired by the **Mobile Actions** feature of [Google AI Edge Gallery](https://play.google.com/store/apps/details?id=com.google.ai.edge.gallery).

---

## Screenshots / Feature overview

| Chat Tab | Model Tab | Settings Tab |
|---|---|---|
| Text + audio input | Model load / unload | HF model ID input |
| FunctionGemma tool cards | Live status & memory | Parameter sliders |
| Multi-turn history | Preset model shortcuts | System prompt editor |

---

## Requirements

| Requirement | Version |
|---|---|
| Python | 3.10.x (pyenv `py104mobileapp`) |
| Kivy | 2.3.1 (pre-installed) |
| KivyMD | 1.2.0 (pre-installed) |
| PyTorch | 2.11.0 (pre-installed) |
| transformers | ≥ 4.40.0 |
| accelerate | ≥ 0.27.0 |
| sounddevice | ≥ 0.4.6 *(audio recording)* |
| numpy | ≥ 1.24.0 |
| SpeechRecognition | ≥ 3.10.0 *(STT)* |

---

## Three-tab UI

### Tab 1 — Chat

The main conversation screen.

- **Status bar** at the top shows the currently loaded model name (green) or an error/idle indicator.
- **Chat history** scrolls automatically; user messages appear on the right, model responses on the left.
- **FunctionCallBubble** cards appear inline whenever FunctionGemma calls a tool — showing the function name, arguments passed, and the returned result.
- **Thinking spinner** appears while the model is generating.
- **Text input** with "send on Enter" support.
- **Microphone button** (hidden when `sounddevice` / `SpeechRecognition` are not installed) records voice and submits the transcribed text automatically.
- **Clear conversation** button in the toolbar wipes the history.

### Tab 2 — Model

The model management screen.

- **Status card** shows model name, device, parameter count, and GPU memory.
- **Indeterminate progress bar** animates while the model is loading.
- **Unload** button frees memory (disabled when no model is loaded).
- **Recommended models** list — tap any row to load that model immediately.
- **Available Functions** list — shows every tool registered in `FunctionExecutor`.

### Tab 3 — Settings

All configurable parameters in one place.

| Section | Control | Description |
|---|---|---|
| Model | Text field | HuggingFace model ID (any `AutoModelForCausalLM`-compatible repo) |
| Model | Text field | Device: `auto` / `cpu` / `cuda` / `mps` |
| Model | Text field | Torch dtype: `float16` / `bfloat16` / `float32` |
| Inference | Slider (64–2048) | Max new tokens |
| Inference | Slider (0.05–2.0) | Temperature |
| Inference | Slider (0.05–1.0) | Top-P |
| Inference | Slider (1–200) | Top-K |
| Inference | Slider (1.0–2.0) | Repetition penalty |
| Inference | Toggle | Stochastic sampling vs greedy |
| Chat | Toggle | Enable / disable system prompt |
| Chat | Multi-line text field | System prompt text |

Pressing **Save Settings** persists all values to `config.json` immediately.
Pressing **Load Model** saves and then triggers `ModelManager.load_model()`.
The **↺ Reset** toolbar button restores all factory defaults.

---

## FunctionGemma mode

When a model whose ID contains `functiongemma`, `function-gemma`, or
`function_gemma` is loaded, the engine enters **FunctionGemma mode**:

1. All schemas from `FunctionExecutor` are serialised to JSON and injected into the prompt before the user's message.
2. The model outputs a `<function_call>{"name": "…", "arguments": {…}}</function_call>` tag.
3. `InferenceEngine.parse_function_call()` extracts the JSON.
4. `FunctionExecutor.execute()` runs the matching Python callable.
5. A `FunctionCallBubble` card is added to the chat showing name, args, and result.

### Built-in tools

| Function | Description |
|---|---|
| `get_current_time` | Returns the current local time (HH:MM:SS) |
| `get_current_date` | Returns today's date as a readable string |
| `calculate` | Evaluates a safe math expression (`sqrt`, `sin`, `cos`, `log`, etc.) |
| `get_weather` | Returns mock weather data for a given city |
| `web_search` | Returns mock search results for a query |
| `set_reminder` | Schedules a reminder N minutes from now |

> **Note:** `get_weather` and `web_search` return *simulated* data.  
> Replace their implementations in `app/core/function_executor.py` with real API calls if needed.
>
> **Tool return format (required):** every function result must be a dictionary in this exact shape:
> `{"ToolResponseForAssistant": "ALL*COMMENTS*TO*THE*ASSISTANT"}`.
> This applies to built-in, global custom, and model-specific functions.

---

## Audio input

Audio recording requires two optional packages:

```bash
pip install sounddevice numpy SpeechRecognition
```

Three transcription backends are supported (configurable in Settings → `config.json`):

| Method | Key | Requires internet | Notes |
|---|---|---|---|
| Google Web Speech API | `"google"` | Yes | Default; good accuracy |
| CMU Sphinx | `"sphinx"` | No | Requires `pocketsphinx` |
| OpenAI Whisper | `"whisper"` | No | Requires `openai-whisper` |

---

## Adding a custom function

```python
from app.core.function_executor import FunctionExecutor, ParameterSchema

executor = FunctionExecutor()

def my_tool(city: str) -> dict:
    return {"ToolResponseForAssistant": f"Custom result for {city}"}

executor.register(
    name        = "my_tool",
    description = "Does something useful with a city name.",
    parameters  = {
        "city": ParameterSchema(
            param_type  = "string",
            description = "Name of the city.",
        )
    },
    fn = my_tool,
)
```

After registration the new function's schema is automatically included in the
FunctionGemma prompt on the next request. Keep function returns in dictionary
format: `{"ToolResponseForAssistant": "<message>"}`.

---

## Configuration file (`config.json`)

Generated automatically on first launch.  Edit manually or use the Settings tab.

```json
{
  "model": {
    "model_id": "google/gemma-2b-it",
    "device": "auto",
    "torch_dtype": "float16"
  },
  "inference": {
    "max_new_tokens": 512,
    "temperature": 0.7,
    "top_p": 0.9,
    "top_k": 50,
    "repetition_penalty": 1.1,
    "do_sample": true
  },
  "chat": {
    "system_prompt": "You are a helpful AI assistant.",
    "use_system_prompt": true,
    "max_history_turns": 0
  },
  "audio": {
    "transcription_method": "google",
    "language": "en-US"
  },
  "ui": {
    "theme_style": "Dark",
    "primary_palette": "DeepPurple"
  }
}
```

---

## Logging

One log file per module per day is written to the `logs/` directory:

```
logs/FuncGemmaChat_20260420.log
logs/ModelManager_20260420.log
logs/InferenceEngine_20260420.log
...
```

Console output shows INFO-level messages; files capture DEBUG-level detail.

---


#### 4. Optional — app log files on device

If the app starts far enough to write logs, Kivy and the app logger may create files under the app’s private storage. Package name: `org.earthisgreen.funcgemmachat`.

```bash
adb shell run-as org.earthisgreen.funcgemmachat ls -la files/app/logs 2>/dev/null
adb shell run-as org.earthisgreen.funcgemmachat ls -la files/app/.kivy/logs 2>/dev/null
```

(`run-as` works on debug builds signed with a debug key.)

### Android Python dependencies (`buildozer.spec`)

Buildozer / python-for-android **does not install pip transitive dependencies** automatically. Every import used at runtime must appear in `requirements` in `buildozer.spec`.

| Layer | Packages |
|-------|----------|
| **UI** | `kivy`, `kivymd`, `filetype`, `pillow` |
| **HTTP / Hub download** | `requests`, `certifi`, `urllib3`, `charset-normalizer`, `idna`, `httpx`, `httpcore`, `h11`, `anyio`, `sniffio`, `filelock`, `fsspec`, `tqdm` |
| **HuggingFace Hub** | `huggingface_hub`, `packaging`, `typing_extensions` |
| **Transformers / tokenizer** | `transformers`, `tokenizers`, `safetensors`, `numpy`, `pyyaml`, `regex`, `typer`, `click`, `rich`, `annotated-doc`, `shellingham` |
| **On-device inference** | `ai_edge_litert`, `litert-lm-api-nightly`, `litert-lm` |

**`regex` / `transformers` on Android:** p4a ships **`regex==2019.6.8`**. Newer **`transformers`** rejects that at import. The APK pins **`transformers==5.2.0`** when possible, and `model_manager.py` applies an Android **metadata shim** so version checks see a compatible regex. Rebuild after changing Python code: `./scripts/docker_android_debug.sh`.

If the app crashes or model load fails with `No module named '…'` or a version mismatch, add the missing name to `requirements` and rebuild. Check logcat (section above) for the exact module name.

### Notes

- All `android.*` settings live under `[app]` in `buildozer.spec` — there is no
  separate `[android]` section in buildozer's schema; keys placed there are
  silently ignored, and buildozer falls back to its defaults.
- Build artifacts are excluded via `.gitignore` (`.buildozer/`, `bin/`).
- Desktop-only packages (not in the Android APK): `torch`, `accelerate`, `sounddevice`, `SpeechRecognition`, etc. See `requirements.txt`.

---


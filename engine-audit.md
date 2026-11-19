# Voxen AI Engine Audit

- **Purpose**: Document the runtime port, AI model in use, and whether Flask is used in the AI engine.
- **Scope**: `/Users/tonwhan/Desktop/Voxen/ai-engine` (ignoring `venv/` and `wandb/` dependency noise).
- **Date**: 2026-11-19
- **Files reviewed**: `app.py` (603 lines), `finetune.py` (116 lines), `chat_test.py` (71 lines), `requirements.txt`, `dataset.jsonl`.

---

## 1. What port is the AI running on?

**Answer**: The AI server runs on **port 8000**, bound to host `0.0.0.0`.

**Evidence**:
- `ai-engine/app.py` lines 14-15: `AI_ENGINE_HOST` / `AI_ENGINE_PORT` read from env (defaults `0.0.0.0:8000`).
- `ai-engine/app.py` lines 674-675:
  ```python
  print(f"🚀 Starting Voxen API Server on {AI_ENGINE_HOST}:{AI_ENGINE_PORT}...")
  uvicorn.run(app, host=AI_ENGINE_HOST, port=AI_ENGINE_PORT)
  ```
- `ai-engine/app.py` line 599: Gradio UI is mounted on the same FastAPI app via `gr.mount_gradio_app(app, demo, path="/")`, so Gradio is served on the same port (8000), not a separate one.

**Notes**: The server is FastAPI + uvicorn, not Flask/Flask's built-in server. Port 8000 is configurable via `AI_ENGINE_PORT` env var.

---

## 2. What AI model is this project using?

**Answer**: The base model is **Qwen2.5-7B-Instruct** (Hugging Face: `Qwen/Qwen2.5-7B-Instruct`) when run from source, or a local `./models/qwen3-8b` if that directory exists. A fine-tuned **LoRA adapter** is loaded from `./models/qwen3-8b-voxen` for CAD JSON generation.

**Evidence**:
- `ai-engine/app.py` lines 17-21: model path resolution logic with env override `BASE_MODEL_PATH` / `FINETUNED_PATH`.
- `ai-engine/app.py` lines 81-100: if `FINETUNED_PATH` exists, the base model is wrapped with `peft.PeftModel.from_pretrained(base_model, FINETUNED_PATH)`.
- `ai-engine/finetune.py` lines 10-13: same model resolution logic for fine-tuning.
- `ai-engine/finetune.py`: LoRA config with `r=32`, `lora_alpha=64`, target modules `q/v/k/o/gate/up/down_proj`, `task_type=CAUSAL_LM`, trained with `trl.SFTTrainer` on `dataset.jsonl`.
- `ai-engine/chat_test.py` line 4: `MODEL_PATH = "./models/qwen3-8b"` with comment "Loading Qwen3-8B on AMD MI300X".

**Notes**: Two model names appear in the codebase (`qwen3-8b` local path vs `Qwen2.5-7B-Instruct` HF repo id). The local path is preferred when present; otherwise the HF model is used. The system prompt (`app.py` lines 38-75) instructs the model to output CAD JSON only. Fine-tuning targets CAD JSON generation via LoRA on `dataset.jsonl` (2 samples).

Provider selection is controlled by `USE_CLOUD_AI` env var:
- `USE_CLOUD_AI=false` (default): local transformers model.
- `USE_CLOUD_AI=true`: OpenRouter API (`LLM_API_BASE_URL`, `LLM_API_KEY`, `LLM_MODEL`).

---

## 3. Is Flask used on this project?

**Answer**: **No.** Flask is not used anywhere in the `ai-engine` source.

**Evidence**:
- `ai-engine/requirements.txt` (8 lines total): `transformers`, `torch`, `peft`, `cadquery==2.4.0`, `accelerate`, `fastapi`, `uvicorn`, `gradio` — **no `flask` entry**.
- Grep for `flask` across `ai-engine/` returned **no matches**.
- `ai-engine/app.py` imports: `transformers`, `torch`, `gradio`, `fastapi`/`uvicorn` (via `uvicorn.run`), `peft` — no Flask import.

**Notes**: `starlette` may appear only as a transitive dependency inside `venv/` (FastAPI/uvicorn), which was explicitly excluded from scope. It is not Flask and not a direct dependency of ai-engine.

---

## Stack Summary

| Layer | Technology |
|-------|------------|
| Web framework | **FastAPI** (`app.py`) |
| ASGI server | **uvicorn** on `0.0.0.0:8000` (`app.py:675`) |
| UI | **Gradio** mounted at path `/` (`app.py:599`) |
| Model runtime | **transformers** + **torch** (bfloat16, `device_map="auto"`) |
| Fine-tuning | **peft** LoRA + **trl** `SFTTrainer` on `dataset.jsonl` (`finetune.py`) |
| Base model | `Qwen/Qwen2.5-7B-Instruct` (HF) or local `./models/qwen3-8b` |
| Fine-tuned adapter | `./models/qwen3-8b-voxen` (LoRA, CAUSAL_LM) |
| Output format | CAD JSON per system prompt (`app.py:38-75`) |
| 3D rendering | CAD JSON rendered in a **Three.js iframe** (frontend, out of scope for ai-engine) |
| STEP export | **cadquery==2.4.0** (`/export_step` endpoint, `app.py`) |

**Flask usage**: None. Stack is FastAPI/uvicorn/Gradio, not Flask.

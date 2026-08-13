
# SupoClip — Fork [For Other Details go to main repo ]

---

# SupoClip — Updates Log
 
This document tracks the configuration and code changes made to this SupoClip deployment, in the order they were made. For each item: what changed, why, which file(s), and whether a rebuild was required.
 
---
 
## 1. Custom transcript-analysis system prompt (file-based, env-configurable)
 
**What changed:** The LLM system prompt used for clip/segment selection can now be loaded from an external `.txt` file instead of only the hardcoded string in `ai.py`.
 
**Why:** Lets the prompt be tuned/iterated without touching Python source, and makes prompt changes independently trackable.
 
**Files:**
- `backend/src/config.py` — added `transcript_system_prompt_path` (reads `TRANSCRIPT_SYSTEM_PROMPT_PATH`)
- `backend/src/ai.py` — added `_load_transcript_system_prompt()`; falls back to the built-in default prompt if the env var is unset or the file can't be read; loader moved into `get_transcript_agent()` for hot-reload on config change
- `docker-compose.yml` — mounted a `./prompts` volume into `backend` and `worker` services
- `.env` — added `TRANSCRIPT_SYSTEM_PROMPT_PATH=/app/prompts/transcript_system_prompt.txt`
**Rebuild:** Yes (Python source + dependency wiring change).
 
---
 
## 2. Engineered upgrade to the transcript-analysis prompt itself
 
**What changed:** Replaced the flat, repetitive original prompt with a restructured version: non-negotiable rules surfaced first, a "cold viewer test" for segment quality, calibrated virality-scoring anchors (fixes score-clustering in the 15–19 band), an explicit pre-output self-check for timestamp/duration validity, and a literal JSON schema skeleton (improves reliability on smaller/local models).
 
**Why:** Original prompt produced inconsistent virality scores and occasional invalid timestamp/duration output; new version tightens both without changing the underlying JSON schema (fully backward-compatible with existing Pydantic models).
 
**File:** `prompts/transcript_system_prompt.txt` (loaded via item #1's mechanism)
 
**Rebuild:** No — picked up via the file-loader from item #1; requires only a worker restart (or is hot-reloaded per-request depending on how item #1 was finished).
 
---

 
## 3. AssemblyAI API compatibility fixes
 
**What changed:** Two breaking changes on AssemblyAI's side were patched:
 
1. **Enum validation error** (`universal-3-pro` not a valid `speech_model` enum member) — normalized via `_assemblyai_speech_model_value()` in `video_utils.py`, mapping requested model names to valid SDK enum values.
2. **Deprecated parameter rejected by the API itself** — AssemblyAI now requires the plural `speech_models` (list) instead of singular `speech_model` (string). Fixed in `get_video_transcript()`:
```python
   config_obj = aai.TranscriptionConfig(
       speaker_labels=True,
       punctuate=True,
       format_text=True,
       speech_models=[speech_model_value],   # was: speech_model=speech_model_value
   )
```
 
**File:** `backend/src/video_utils.py`
 
**Also updated:** `FAST_MODE_TRANSCRIPT_MODEL` default pointed at AssemblyAI's current flagship model (`universal-3-5-pro`) rather than the deprecated generic fallback.
 
**Rebuild:** Yes (Python source change; also required an `assemblyai` SDK version bump in `backend/pyproject.toml`).
 
---
 
---
 
## 4. Adaptive / variable clip count (was hardcoded to 2–5)
 
**What changed:** Clip count no longer hardcoded to a fixed "2-5 segments" — it now scales with source video length/content density, with a configurable safety ceiling.
 
**Details:**
- `build_transcript_analysis_prompt()` now computes a target range from transcript duration instead of a fixed number.
- The persistent system prompt's hardcoded "Find 2-5 compelling segments" line was reworded to instruct content-driven scaling instead of a fixed count.
- `video_service.py`'s truncation logic (previously only applied in `fast` mode via `FAST_MODE_MAX_CLIPS`) now also enforces `MAX_CLIPS` as a hard backstop in `balanced`/`quality` modes (previously `MAX_CLIPS` was set in config but never actually enforced anywhere).
**Files:** `backend/src/ai.py`, `backend/src/services/video_service.py`
 
**Config:**
```
DEFAULT_PROCESSING_MODE=balanced   # or quality — avoids the old fast-mode 4-clip cap
FAST_MODE_MAX_CLIPS=15             # raised from default of 4, only applies in fast mode
MAX_CLIPS=20                       # now actually enforced as the hard ceiling
```
 
**Rebuild:** Yes (Python source change).
 
---
 
## 5. Upload size limit raised (was hardcoded to 1 GB on both frontend and backend)
 
**What changed:** The 1 GB upload cap was found to be two independent hardcoded constants — one client-side (fails fast before upload starts), one server-side (the real enforcement, returns HTTP 413). Both needed updating; fixing only the frontend still resulted in a backend 413.
 
**Files:**
- `frontend/src/components/home-app.tsx` — `MAX_VIDEO_UPLOAD_BYTES` now reads from `NEXT_PUBLIC_MAX_VIDEO_UPLOAD_MB` instead of being hardcoded
- `backend/src/api/routes/media.py` — upload route's size check now reads from `runtime_config.max_upload_bytes`
- `backend/src/config.py` — added `max_upload_bytes` (reads `MAX_UPLOAD_MB`)
**Config:**
```
NEXT_PUBLIC_MAX_VIDEO_UPLOAD_MB=4096
MAX_UPLOAD_MB=4096
```
 
**Note:** This constant was a UX/resource-usage guardrail chosen by the original author, not derived from any external infrastructure limit (no proxy, CDN, or AssemblyAI-side constraint was found to require 1 GB specifically).
 
**Rebuild:** Yes for both services (`frontend` — `NEXT_PUBLIC_*` build-time bake-in; `backend` — Python source change).
 
---
 
## Rebuild summary
 
```bash
docker-compose build --no-cache backend worker frontend
docker-compose up -d
docker-compose logs -f worker
```



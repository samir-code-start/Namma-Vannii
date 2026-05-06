# 🤖 Namma Vanni — AI Coding Handoff Prompt (Gemini)

> Paste this entire prompt to Gemini 2.5 Pro before sharing any files.
> Then attach `engine.py` and `app.py` and say: **"Start with Step 1."**

---

## 🧠 Project Context

You are continuing development on **Namma Vanni** — a voice-to-voice AI civic helpline for Karnataka's 1092 emergency line. The app is built with:

- **`engine.py`** — AI pipeline: Speech-to-Text (Sarvam API), LLM analysis (Groq/LLaMA), TTS (Sarvam + Edge TTS fallback), feedback logging
- **`app.py`** — Streamlit frontend: multi-stage call flow (record → verify → agent dashboard → handover)

A previous AI assistant (Claude Opus) has already completed **Phases 1–5 and Phases 7–8** of the implementation plan. **Only Phase 6 (3 steps) remains.**

---

## ✅ What is Already Done — DO NOT TOUCH

| Phase | Feature | Status |
|-------|---------|--------|
| 1 | `"fear"` sentiment added to engine + app | ✅ Done |
| 2 | Dialect awareness + civic taxonomy in system prompt | ✅ Done |
| 3 | `"partial"` intent in verify flow (`partial_refine` stage) | ✅ Done |
| 4 | Sarvam TTS (primary) + Edge TTS (fallback) | ✅ Done |
| 5 | Verification prompt translated to citizen's language | ✅ Done |
| 6 | Dual transcript display | ⚠️ **INCOMPLETE — your job** |
| 7 | Retry with context (`re_record` stage) | ✅ Done |
| 8 | Confidence-weighted feedback in `feedback.csv` | ✅ Done |

---

## 🎯 Your Task — Phase 6: Dual Transcript Display (3 Steps Only)

The goal: when a citizen speaks in Kannada/Hindi, the agent dashboard should show **both** the original-language transcript AND the English translation side by side.

**Current broken state:**
- `transcribe_audio()` returns a 2-tuple: `(english_text, lang)`
- `_sarvam_stt_original()` exists in `engine.py` but is only called inside `process_audio()` — its result is stored in `ai_data["original_text"]`
- The `agent_ready` stage in `app.py` does **NOT** display `original_text` — only `raw_text` (English)

---

## 📋 Step-by-Step Instructions

### STEP 1 of 3 — Fix `transcribe_audio()` return signature

**File:** `engine.py`
**Function:** `transcribe_audio(audio_path: str)`

**Current signature:**
```python
def transcribe_audio(audio_path: str) -> tuple[str, str]:
    """Returns (english_text, detected_language_code)."""
```

**Change to:**
```python
def transcribe_audio(audio_path: str) -> tuple[str, str, str]:
    """Returns (english_text, detected_language_code, original_transcript).
    
    original_transcript: raw text in citizen's language via Sarvam /speech-to-text (non-translate).
    Falls back to empty string if Sarvam STT original fails.
    """
```

**Logic inside the function:**
1. Keep all existing STT-translate logic (returns `english_text`, `raw_lang`) — do NOT change it
2. After getting `english_text`, call `_sarvam_stt_original(audio_path)` to get `original_text`
3. Return `(english_text, raw_lang, original_text)` as a 3-tuple
4. In MOCK_MODE: return `("The road in our village is very bad, please send officials", "kn", _MOCK_TRANSCRIPT)`

> `_MOCK_TRANSCRIPT` is already defined at line ~81 in engine.py as the Kannada string.

> `_sarvam_stt_original()` is already implemented in engine.py — just call it.

**After this step, stop. Show me the changed function only. Do not proceed to Step 2.**

---

### STEP 2 of 3 — Update all callers of `transcribe_audio()`

**Files:** `engine.py` and `app.py`

There are 3 places that call `transcribe_audio()`. Update each to handle the new 3-tuple:

**In `engine.py` → `process_audio()` (around line 532):**
```python
# BEFORE:
raw_text, lang = transcribe_audio(audio_path)

# AFTER:
raw_text, lang, original_text = transcribe_audio(audio_path)
```
> Remove the separate `_sarvam_stt_original()` call that follows (around line 536) — it's now inside `transcribe_audio()` itself. Keep the `original_text` variable.

**In `app.py` → `verify` stage (around line 266):**
```python
# BEFORE:
raw_text, _ = transcribe_audio("confirm.wav")

# AFTER:
raw_text, _, _ = transcribe_audio("confirm.wav")
```

**In `app.py` → `partial_refine` stage (around line 321):**
```python
# BEFORE:
correction_text, _ = transcribe_audio("correction.wav")

# AFTER:
correction_text, _, _ = transcribe_audio("correction.wav")
```

**In `app.py` → `re_record` stage (around line 364):**
```python
# BEFORE:
retry_text, _ = transcribe_audio("retry.wav")

# AFTER:
retry_text, _, _ = transcribe_audio("retry.wav")
```

**After this step, stop. Show me all changed lines. Do not proceed to Step 3.**

---

### STEP 3 of 3 — Display dual transcript in `agent_ready` stage

**File:** `app.py`
**Stage block:** `agent_ready` (starts around line 393)

Find this existing block (around line 419–425):
```python
raw_transcript = data.get("raw_text", "")
st.markdown(f'''<div style="background: var(--bg-surface); border: 1px solid var(--border-subtle); border-radius: 12px; padding: 20px; margin-top: 16px;">
    <span class="section-label">📄 Raw Transcript</span>
    <p style="font-family: 'Space Mono', monospace; font-size: 14px; line-height: 1.5; color: var(--text-secondary) !important; margin-top: 8px; white-space: pre-wrap; word-break: break-word;">
        {raw_transcript}
    </p>
</div>''', unsafe_allow_html=True)
```

**Replace it with this dual-transcript display:**
```python
raw_transcript = data.get("raw_text", "")
original_transcript = data.get("original_text", "")
lang = data.get("language", "en")
lang_label = {"kn": "ಕನ್ನಡ (Kannada)", "hi": "हिन्दी (Hindi)"}.get(lang, lang.upper())

if original_transcript:
    # Side-by-side: original language + English
    st.markdown(f'''<div style="display:grid;grid-template-columns:1fr 1fr;gap:12px;margin-top:16px;">
        <div style="background:var(--bg-surface);border:1px solid var(--border-subtle);border-left:3px solid var(--accent-blue);border-radius:12px;padding:20px;">
            <span class="section-label">🗣️ Original ({lang_label})</span>
            <p style="font-family:'Space Mono',monospace;font-size:13px;line-height:1.6;color:var(--text-secondary) !important;margin-top:8px;white-space:pre-wrap;word-break:break-word;">
                {original_transcript}
            </p>
        </div>
        <div style="background:var(--bg-surface);border:1px solid var(--border-subtle);border-left:3px solid var(--accent-green);border-radius:12px;padding:20px;">
            <span class="section-label">📄 English Translation</span>
            <p style="font-family:'Space Mono',monospace;font-size:13px;line-height:1.6;color:var(--text-secondary) !important;margin-top:8px;white-space:pre-wrap;word-break:break-word;">
                {raw_transcript}
            </p>
        </div>
    </div>''', unsafe_allow_html=True)
else:
    # Fallback: single transcript (English only)
    st.markdown(f'''<div style="background:var(--bg-surface);border:1px solid var(--border-subtle);border-radius:12px;padding:20px;margin-top:16px;">
        <span class="section-label">📄 Raw Transcript</span>
        <p style="font-family:'Space Mono',monospace;font-size:14px;line-height:1.5;color:var(--text-secondary) !important;margin-top:8px;white-space:pre-wrap;word-break:break-word;">
            {raw_transcript}
        </p>
    </div>''', unsafe_allow_html=True)
```

**After this step, stop. Show me the changed block only. This completes Phase 6.**

---

## ⚠️ Critical Rules for Gemini

1. **One step at a time.** Complete exactly one numbered step, then stop and wait.
2. **Show only changed code.** Don't rewrite entire files. Show diffs or the specific function/block.
3. **Don't touch anything outside the step.** Phases 1–5, 7, 8 are fully working — breaking them will break the demo.
4. **Preserve all CSS variables** (`var(--bg-surface)`, `var(--accent-blue)`, etc.) — the design system is already defined in `init_global_styles()`.
5. **No new dependencies.** Everything needed is already imported.
6. **If unsure about a line number**, search for the exact string in the file rather than guessing by line number.

---

## ✅ Verification Checklist (After All 3 Steps)

Run `streamlit run app.py` and test:
- [ ] Record Kannada speech → process → reach `agent_ready`
- [ ] Agent dashboard shows TWO transcript boxes side-by-side
- [ ] Left box: Kannada original text
- [ ] Right box: English translation
- [ ] If Sarvam STT original fails → gracefully falls back to single English box (no crash)
- [ ] `feedback.csv` still logs correctly after closing a call

---

*Handoff prepared by Claude Sonnet 4.6 | Namma Vanni — AI Bharat Hackathon*

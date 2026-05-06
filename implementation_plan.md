# 🚀 Namma Vanni — Logic Implementation Plan

> Ordered by **most critical first** → least critical last.  
> Each phase is broken into discrete, testable steps.

---

## Phase 1: Add "Fear" Sentiment + Fix Guardrails ⏱️ ~5 min

**Why first:** Problem statement **explicitly lists 6 emotions**: distress, urgency, anger, **fear**, confusion, calm. Current code only has 5. This is a **direct compliance gap** — easiest to fix, highest ROI.

### Step 1.1 — Add `"fear"` to valid sentiments set
- **File:** [engine.py](file:///d:/Ai%20Bahrat%20hackathon/engine.py#L140)
- **Change:** Add `"fear"` to `_VALID_SENTIMENTS`
```diff
-_VALID_SENTIMENTS = {"calm", "confused", "urgent", "distressed", "angry"}
+_VALID_SENTIMENTS = {"calm", "confused", "urgent", "distressed", "angry", "fear"}
```

### Step 1.2 — Add `"fear"` to the LLM system prompt
- **File:** [engine.py](file:///d:/Ai%20Bahrat%20hackathon/engine.py#L102)
- **Change:** Update the `sentiment` field in the JSON schema to include `fear`
```diff
-  "sentiment": "calm|confused|urgent|distressed|angry",
+  "sentiment": "calm|confused|urgent|distressed|angry|fear",
```

### Step 1.3 — Add `"fear"` as a handover trigger
- **File:** [engine.py](file:///d:/Ai%20Bahrat%20hackathon/engine.py#L176)
- **Change:** Add `"fear"` alongside `"distressed"` and `"angry"` in guardrails
```diff
-    elif sentiment in {"distressed", "angry"} or ...
+    elif sentiment in {"distressed", "angry", "fear"} or ...
```

### Step 1.4 — Add `"fear"` UI pill to the app
- **File:** [app.py](file:///d:/Ai%20Bahrat%20hackathon/app.py#L95-L101)
- **Change:** Add fear entry to `SENTIMENT_MAP` and fear CSS class
```diff
 SENTIMENT_MAP = {
     ...
     "angry": ("emotion-angry", "🟥", "Angry"),
+    "fear": ("emotion-fear", "😨", "Fear"),
 }
```
- **Also add CSS class** `.emotion-fear` in `init_global_styles()`:
```css
.emotion-fear { background:rgba(138,43,226,0.15); color:#9b59b6 !important; border:1px solid rgba(138,43,226,0.25); }
```

### ✅ Verify
- Run app → analyze a distressed call → confirm `fear` appears in sentiment detection and triggers handover.

---

## Phase 2: Enhanced System Prompt — Dialect Awareness + Civic Taxonomy ⏱️ ~15 min

**Why second:** Requirement #5 (dialect/cultural understanding) is **completely unaddressed**. Adding rich prompt context is zero-risk, high-impact.

### Step 2.1 — Add dialect awareness block to system prompt
- **File:** [engine.py](file:///d:/Ai%20Bahrat%20hackathon/engine.py#L85-L110)
- **Change:** Append the following after the `GUARDRAILS` section in `_SYSTEM_PROMPT`:

```python
DIALECT AWARENESS (Karnataka):
- North Karnataka dialects: "enu" → "ēnu" (what), "barri" → "banni" (come)
- Bangalore Urban: Code-mixed Kannada-English ("current hogide" = power cut)
- Old Mysuru: Formal Kannada with "appa/amma" honorifics
- Hindi-belt migrants: Hinglish mixed with Kannada words
- Common civic expressions:
  "current hogide" = power cut
  "neer bandilla" = no water supply
  "gutter overflow" = drainage blockage
  "kasa collect aagilla" = garbage not collected
```

### Step 2.2 — Add civic issue taxonomy to system prompt
- **File:** [engine.py](file:///d:/Ai%20Bahrat%20hackathon/engine.py#L85-L110)
- **Change:** Append issue categories after dialect block:

```python
ISSUE CATEGORIES (Karnataka 1092):
- ROAD: potholes, damaged roads, flooding, waterlogging
- WATER: supply disruption, contamination, leakage, bore well
- ELECTRICITY: power cuts, street lights, transformer failure
- GARBAGE: collection missed, illegal dumping, burning
- DRAINAGE: overflow, blockage, sewage leak
- SAFETY: crime, harassment, emergency, accidents
- GOVERNMENT: corruption, missing services, documentation issues
```

### ✅ Verify
- Feed a Kannada audio about "current hogide" → confirm LLM correctly maps to ELECTRICITY/power cut.

---

## Phase 3: Add "Partially Correct" Intent to Verification Flow ⏱️ ~20 min

**Why third:** Problem statement says capture: **correct / partially correct / incorrect**. Current system only has Yes/No/Unclear. This is a **judging criteria gap**.

### Step 3.1 — Add partial tokens to `parse_confirmation()` in engine.py
- **File:** [engine.py](file:///d:/Ai%20Bahrat%20hackathon/engine.py#L197-L228)
- **Change:** Add a `partial_tokens` list and detection logic **before** the yes/no checks:

```python
partial_tokens = ["but", "almost", "mostly", "partially", "partly", 
                  "half", "kinda", "sort of", "not fully", "not exactly",
                  "haan par", "haan lekin", "sari aadre", "yes but"]

# Check partial FIRST (because "yes but..." should be partial, not confirmed)
partial_hits = [w for w in partial_tokens if w in t]
if len(partial_hits) > 0 and len(yes_hits) > 0:
    return {"intent": "partial", "summary": t[:80]}
```

### Step 3.2 — Add partial tokens to `parse_smart_confirmation()` in app.py
- **File:** [app.py](file:///d:/Ai%20Bahrat%20hackathon/app.py#L117-L136)
- **Change:** Mirror the partial detection logic:

```python
partial_phrases = ["but", "almost", "mostly", "partially", "not exactly",
                   "not fully", "haan par", "sari aadre", "yes but"]
partial_hits = [w for w in partial_phrases if w in t]

if len(partial_hits) > 0 and len(pos_hits) > 0:
    return {"intent": "partial"}
```

### Step 3.3 — Update LLM intent extraction prompt
- **File:** [engine.py](file:///d:/Ai%20Bahrat%20hackathon/engine.py#L219)
- **Change:** Update the system prompt for LLM intent extraction to include `"partial"`:
```diff
-"intent":"confirmed"|"denied"|"unclear"
+"intent":"confirmed"|"denied"|"partial"|"unclear"
```

### Step 3.4 — Handle `"partial"` intent in app.py verify stage
- **File:** [app.py](file:///d:/Ai%20Bahrat%20hackathon/app.py#L250-L263)
- **Change:** Add a new branch for `partial` between `confirmed` and `denied`:

```python
elif result["intent"] == "partial":
    # Keep existing context, re-record for refinement
    st.info("🔄 Partially understood. Please clarify what was different.")
    st.session_state.stage = "partial_refine"
    st.rerun()
```

### Step 3.5 — Create `partial_refine` stage in app.py
- **File:** [app.py](file:///d:/Ai%20Bahrat%20hackathon/app.py) (new stage block)
- **Change:** Add a new stage that:
  1. Shows what the AI understood
  2. Lets the citizen record a correction
  3. Sends correction to LLM with context: *"Previous analysis was partially correct. Citizen's correction: '...'. Refine."*
  4. Re-routes to `verify` with updated `ai_data`

### Step 3.6 — Add `"partial_refine"` to session state defaults
- **File:** [app.py](file:///d:/Ai%20Bahrat%20hackathon/app.py#L141)
- **Change:** No default change needed (it's a stage value, not a key), but ensure `_DEFAULTS` doesn't break.

### Step 3.7 — Update feedback logging for partial responses
- **File:** [engine.py](file:///d:/Ai%20Bahrat%20hackathon/engine.py#L384-L407)
- **Change:** `citizen_response` field should accept `"Partial"` value alongside `"Confirmed"` and `"Handover"`.

### ✅ Verify
- Record "yes but the location is wrong" → confirm app routes to `partial_refine` stage.
- Record correction → confirm AI refines its analysis.

---

## Phase 4: Replace Edge TTS with Sarvam TTS ⏱️ ~25 min

**Why fourth:** Problem statement says *"voice-to-voice in the citizen's language"*. Edge TTS speaks Indic languages with a Microsoft accent. Sarvam TTS has **native Indian voices** — core differentiator.

### Step 4.1 — Add Sarvam TTS API helper function
- **File:** [engine.py](file:///d:/Ai%20Bahrat%20hackathon/engine.py) (new function)
- **Change:** Create `_sarvam_tts(text, lang_code)` that calls:
```
POST https://api.sarvam.ai/text-to-speech
Header: api-subscription-key: <SARVAM_API_KEY>
Body: {
    "inputs": [text],
    "target_language_code": "kn-IN" | "hi-IN" | "en-IN",
    "speaker": "meera",  # or "arvind" for male
    "model": "bulbul:v2"
}
```
- Returns: audio bytes (base64 decoded)

### Step 4.2 — Add Sarvam language code mapping
- **File:** [engine.py](file:///d:/Ai%20Bahrat%20hackathon/engine.py#L60-L66)
- **Change:** Add `SARVAM_TTS_LANG_MAP`:
```python
SARVAM_TTS_LANG_MAP = {
    "kn": "kn-IN",
    "hi": "hi-IN",
    "en": "en-IN",
}
SARVAM_TTS_URL = "https://api.sarvam.ai/text-to-speech"
```

### Step 4.3 — Rewrite `generate_tts()` to use Sarvam primary, Edge fallback
- **File:** [engine.py](file:///d:/Ai%20Bahrat%20hackathon/engine.py#L323-L357)
- **Change:** 
  1. Try Sarvam TTS first
  2. If Sarvam fails, fall back to Edge TTS (existing logic)
  3. Write audio bytes to `verify.mp3`

### Step 4.4 — Remove `edge_tts` as a hard dependency (make it optional fallback)
- **File:** [engine.py](file:///d:/Ai%20Bahrat%20hackathon/engine.py#L13)
- **Change:** Wrap `import edge_tts` in try/except for graceful degradation

### ✅ Verify
- Run app → record issue → listen to verification → confirm voice is native Indian accent, not Microsoft.

---

## Phase 5: Translate Verification Prompt to Citizen's Language ⏱️ ~20 min

**Why fifth:** Currently LLM generates English-only verification. A Kannada citizen hears English playback — this **breaks the core voice-to-voice requirement**.

### Step 5.1 — Add Sarvam translate helper function
- **File:** [engine.py](file:///d:/Ai%20Bahrat%20hackathon/engine.py) (new function)
- **Change:** Create `_sarvam_translate(text, source_lang, target_lang)`:
```
POST https://api.sarvam.ai/translate
Body: {
    "input": text,
    "source_language_code": "en-IN",
    "target_language_code": "kn-IN"
}
```
- Returns translated text string

### Step 5.2 — Update `process_audio()` to translate before TTS
- **File:** [engine.py](file:///d:/Ai%20Bahrat%20hackathon/engine.py#L359-L382)
- **Change:** After `analyze_transcript()`, translate the verification prompt:
```python
verification_prompt = ai_data.get("verification_prompt", "")
citizen_lang = ai_data.get("language", "kn")

if citizen_lang != "en":
    translated_prompt = _sarvam_translate(verification_prompt, "en-IN", f"{citizen_lang}-IN")
    ai_data["verification_prompt_translated"] = translated_prompt
    tts_text = translated_prompt
else:
    tts_text = verification_prompt

tts_path = generate_tts(tts_text, citizen_lang)
```

### Step 5.3 — Display both English and translated prompts in app.py
- **File:** [app.py](file:///d:/Ai%20Bahrat%20hackathon/app.py#L222-L230)
- **Change:** Show the English prompt as text + play translated audio:
```python
# Show English text for agent
st.markdown(f"AI Prompt (English): {base_prompt}")
# Show translated text for context
if "verification_prompt_translated" in data:
    st.markdown(f"AI Prompt ({lang}): {data['verification_prompt_translated']}")
```

### ✅ Verify
- Record Kannada audio → confirm verification plays back in Kannada, not English.

---

## Phase 6: Dual Transcript Display (Original + English) ⏱️ ~25 min

**Why sixth:** Agent sees what the citizen actually said in their language. **Massive credibility for human-in-the-loop** showcase.

### Step 6.1 — Add Sarvam STT (non-translate) call
- **File:** [engine.py](file:///d:/Ai%20Bahrat%20hackathon/engine.py#L249-L287)
- **Change:** Create `_sarvam_stt_original(audio_path)` that calls:
```
POST https://api.sarvam.ai/speech-to-text
Files: file=audio
Header: api-subscription-key
```
- Returns: original language transcript (e.g., Kannada text)

### Step 6.2 — Update `transcribe_audio()` to return both transcripts
- **File:** [engine.py](file:///d:/Ai%20Bahrat%20hackathon/engine.py#L249-L287)
- **Change:** Call BOTH endpoints:
```python
def transcribe_audio(audio_path: str) -> tuple[str, str, str]:
    """Returns (english_text, detected_language, original_transcript)."""
    # Call speech-to-text-translate → English
    # Call speech-to-text → Original language
    return english_text, lang, original_text
```

### Step 6.3 — Update `process_audio()` to carry original transcript
- **File:** [engine.py](file:///d:/Ai%20Bahrat%20hackathon/engine.py#L359-L382)
- **Change:** Pass `original_text` through the pipeline:
```python
raw_text, lang, original_text = transcribe_audio(audio_path)
final = {**ai_data, "raw_text": raw_text, "original_text": original_text, ...}
```

### Step 6.4 — Update app.py to display dual transcript
- **File:** [app.py](file:///d:/Ai%20Bahrat%20hackathon/app.py#L308-L314) (agent_ready stage)
- **Change:** Show both transcripts in the agent dashboard:
```python
# Original language transcript
original = data.get("original_text", "")
if original:
    st.markdown(f"Original ({lang}): {original}")
# English transcript
st.markdown(f"English: {raw_transcript}")
```

### Step 6.5 — Update all callers of `transcribe_audio()` in app.py
- **File:** [app.py](file:///d:/Ai%20Bahrat%20hackathon/app.py#L246)
- **Change:** Update verify stage to handle 3-tuple return:
```diff
-raw_text, _ = transcribe_audio("confirm.wav")
+raw_text, _, _ = transcribe_audio("confirm.wav")
```

### ✅ Verify
- Record Kannada audio → confirm both Kannada original and English translation appear on agent dashboard.

---

## Phase 7: Retry with Context Preservation ⏱️ ~20 min

**Why seventh:** When citizen says "No" (denied), currently starts over. This loses context. **Demonstrates learning from feedback** (Requirement #3).

### Step 7.1 — Update `denied` handler to preserve context
- **File:** [app.py](file:///d:/Ai%20Bahrat%20hackathon/app.py#L253-L259)
- **Change:** Instead of resetting, keep `ai_data` and re-record:
```python
elif result["intent"] == "denied":
    if st.session_state.attempts >= 2 or data.get("handover"):
        st.session_state.stage = "handover"
    else:
        # Preserve context, go back to re-record
        st.session_state.stage = "re_record"
    st.rerun()
```

### Step 7.2 — Create `re_record` stage
- **File:** [app.py](file:///d:/Ai%20Bahrat%20hackathon/app.py) (new stage block)
- **Change:** New stage that:
  1. Shows previous AI summary (what was wrong)
  2. Lets citizen re-record
  3. Sends to LLM with context: *"Previous attempt was denied. Previous analysis: '...'. Citizen's new recording: '...'. Re-analyze with corrections."*
  4. Routes back to `verify`

### Step 7.3 — Add LLM re-analysis function
- **File:** [engine.py](file:///d:/Ai%20Bahrat%20hackathon/engine.py) (new function)
- **Change:** Create `re_analyze_transcript(new_text, previous_analysis)`:
```python
def re_analyze_transcript(new_text: str, previous_analysis: dict) -> dict:
    """Re-analyze with context from previous denied attempt."""
    context = f"""Previous analysis was DENIED by citizen.
Previous AI summary: '{previous_analysis.get('normalized_issue', '')}'
Citizen's correction/new input: '{new_text}'
Re-analyze incorporating the citizen's feedback."""
    
    messages = [
        {"role": "system", "content": _SYSTEM_PROMPT},
        {"role": "user", "content": context}
    ]
    raw = _sarvam_chat(messages=messages)
    parsed = extract_json(_strip_fences(raw))
    return _enforce_guardrails(parsed)
```

### ✅ Verify
- Record issue → AI misunderstands → say "No" → re-record with correction → confirm AI refines its analysis using previous context.

---

## Phase 8: Confidence-Weighted Feedback Storage ⏱️ ~15 min

**Why last:** This is a data quality improvement that strengthens the "learning from feedback" story (Requirement #3), but doesn't affect real-time user experience.

### Step 8.1 — Add `feedback_weight` column to CSV headers
- **File:** [engine.py](file:///d:/Ai%20Bahrat%20hackathon/engine.py#L54-L58)
- **Change:** Add `"feedback_weight"` to `FEEDBACK_HEADERS`:
```diff
 FEEDBACK_HEADERS = [
     "timestamp", "language", "raw_text", "ai_issue",
     "confidence", "sentiment", "citizen_response",
-    "agent_correction", "handover",
+    "agent_correction", "handover", "feedback_weight",
 ]
```

### Step 8.2 — Calculate weight in `log_feedback()`
- **File:** [engine.py](file:///d:/Ai%20Bahrat%20hackathon/engine.py#L384-L407)
- **Change:** Compute weight based on citizen response + confidence:
```python
response = data.get("citizen_response", "")
confidence = float(data.get("confidence", 0.5))

if response == "Confirmed":
    weight = round(0.5 + (confidence * 0.5), 2)  # 0.5-1.0
elif response == "Partial":
    weight = round(0.3 * confidence, 2)           # 0.0-0.3
elif response == "Handover":
    weight = round(-0.5 * confidence, 2)           # negative signal
else:
    weight = 0.0

row["feedback_weight"] = weight
```

### Step 8.3 — Update app.py feedback calls
- **File:** [app.py](file:///d:/Ai%20Bahrat%20hackathon/app.py#L322-L329)
- **Change:** Ensure `citizen_response` is set correctly for all paths:
  - `"Confirmed"` → agent_ready log
  - `"Partial"` → partial_refine log  
  - `"Handover"` → handover log

### ✅ Verify
- Complete a full call flow → check `feedback.csv` → confirm `feedback_weight` column exists with correct values.

---

## Execution Summary

| Phase | Change | Files | Est. Time | Requirement |
|:-----:|--------|-------|:---------:|:-----------:|
| 1 | Add "fear" sentiment | `engine.py`, `app.py` | 5 min | #6 Sentiment |
| 2 | Dialect + Taxonomy prompt | `engine.py` | 15 min | #5 Dialect |
| 3 | "Partially correct" intent | `engine.py`, `app.py` | 20 min | #2 Verification |
| 4 | Sarvam TTS swap | `engine.py` | 25 min | #1 Voice-to-Voice |
| 5 | Translate verification prompt | `engine.py`, `app.py` | 20 min | #1 Voice-to-Voice |
| 6 | Dual transcript display | `engine.py`, `app.py` | 25 min | #7 Human-in-Loop |
| 7 | Retry with context | `engine.py`, `app.py` | 20 min | #3 Learning |
| 8 | Weighted feedback | `engine.py`, `app.py` | 15 min | #3 Learning |

> [!IMPORTANT]
> **Phases 1–5 alone cover ALL 7 requirements** in the problem statement and can be completed in ~85 minutes.
> Phases 6–8 are polish that strengthen the demo for judges.

> [!TIP]
> After each phase, run `streamlit run app.py` and test end-to-end before moving to the next phase.

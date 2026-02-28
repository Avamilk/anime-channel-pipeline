# Pipeline Impact Analysis — Visual + Tools Registry Updates

## Changes Made

### tools-registry.skill
1. Added `origin` and `prompt_language` fields to every capability
2. Renamed capabilities more explicitly: IMAGE_PRIMARY, IMAGE_FAST, VIDEO_STANDARD, VIDEO_CINEMATIC, VIDEO_FAST
3. Added "Prompt Language Routing" explanation section
4. Added Quality Suffix Constants (zh + en versions, with no-BGM/no-subtitles baked in)
5. Added Camera Movement Reference Library section with file naming conventions
6. Corrected IMAGE_PRIMARY to Seedream (was Flux — Seedream is actually the ByteDance image model, better for anime aesthetic alignment)

### visual.skill
1. Replaced "always English" with language routing logic (read prompt_language → translate if zh)
2. Added 8-dimension prompt formula with explicit ordering
3. "One scene, one thing" rule as enforced check (split multi-beat scenes before generating)
4. No-BGM / no-subtitles baked into quality suffix — no longer optional
5. Dual-language storage: prompt_en + prompt_submitted in every generation_jobs entry
6. Camera reference video library integration
7. Natural language translation guidance (colloquial, not technical jargon)
8. 9-grid storyboard trick as optional technique for multi-beat sequences
9. Character cards now store prompt_fragment_en + prompt_fragment_zh
10. Cam-ref library bootstrap as new pipeline phase

---

## Pipeline Impact by Skill

### Director — MINOR IMPACT
- New optional first-run step: cam_ref_bootstrap (one-time, ~140 credits)
- state.json generation_jobs schema now has extra fields (prompt_en, prompt_submitted, prompt_language)
- No logic changes to orchestration, polling, or handoffs
- Weekly analytics unaffected

### Scriptwriter — ZERO IMPACT  
- Continues writing entirely in English
- scene.visual_description, scene.anime_style_note — unchanged fields
- No new requirements on Scriptwriter
- The translation happens in Visual, invisible to Scriptwriter

### Researcher — ZERO IMPACT
- Research brief format unchanged
- story_tags still drive content routing (unchanged logic)

### Scout — ZERO IMPACT
- No changes affecting story discovery

### Audio — ZERO IMPACT
- No-BGM rule means Audio's own soundtrack layer has a clean slate to work on
- Previously: Assembly had to detect and strip Seedance's auto-generated BGM before compositing Audio's layer
- Now: Seedance doesn't add BGM in the first place → Assembly is simpler

### Assembly — SMALL POSITIVE IMPACT
- No longer needs to strip AI-generated BGM from video clips (Seedance was adding it ~80% of the time)
- Video clips arrive clean → simpler audio compositing pipeline
- One less failure mode (BGM detection/stripping was fragile)

### Resilience — MINOR IMPACT
- Retry logic unchanged in structure
- On prompt rewrite + retry: Visual now re-translates from prompt_en for the new attempt
- This makes retries cleaner — always retranslating from the canonical English, not re-translating a translation
- Fallback to western provider: prompt_language switches to "en" automatically → no translation step

### Publisher — ZERO IMPACT

### Scheduler — ZERO IMPACT

### Dashboard — ZERO IMPACT

### Memory — ZERO IMPACT

---

## State.json Schema Changes

### generation_jobs entries — expanded
Before:
```json
{
  "scene_3": {
    "job_id": "task_xxx",
    "status": "complete",
    "file_path": "assets/videos/scene_3.mp4",
    "prompt_used": "samurai draws sword..."
  }
}
```

After:
```json
{
  "scene_3": {
    "job_id": "task_xxx",
    "status": "complete",
    "file_path": "assets/videos/scene_3.mp4",
    "prompt_en": "Anime cinematic dark seinen style, young samurai...",
    "prompt_submitted": "动漫电影风格，黑暗青年漫，年轻武士...",
    "prompt_language": "zh",
    "provider": "xskill:seedance_2.0_fast"
  }
}
```

### New top-level fields
```json
{
  "cam_ref_library": {
    "push_in_slow": "https://xskill.ai/stable/cam_push_in_slow.mp4",
    ...
  },
  "content_routing_notes": "Routed scene 3 to Runway: story_tag=war",
  "last_credit_check": 847
}
```

### character_cards — expanded
Before: `prompt_fragment` (English only)
After: `prompt_fragment_en` + `prompt_fragment_zh` (both stored at anchor test time)

---

## New Pipeline Phase: Cam-Ref Bootstrap

One-time first-run step Director should call before any video generation:

```
Director checks: does assets/cam-refs/ exist with 7 files?
  YES → skip
  NO → call visual skill with "cam_ref_bootstrap"
       Cost: ~140 credits once
       Time: ~15 minutes (7 async video jobs)
       Benefit: reused across every video forever
```

This is optional — pipeline works without it. Camera moves described in text still work.
But the reference library makes camera control significantly more reliable.

---

## Risk Assessment

### Low risk
- Dual-language storage: additive only, no existing fields removed
- Quality suffix: previously written manually/inconsistently, now standardised
- No-BGM rule: removes a problem (unwanted audio) rather than adding one

### Medium risk — needs monitoring
- Language routing logic: if translation call fails mid-prompt, retry should fall back to English
  → Resilience skill should add: on translation error → log warning, submit English anyway
- Prompt quality in Chinese: if translated prompts perform worse than expected, can revert
  by changing prompt_language back to "en" in tools registry — single file change

### Not changed (explicitly preserved)
- Fallback provider chain logic
- Polling cadence (60s first, 90s interval, 600s timeout for video)
- Credit check threshold
- Content routing by story_tags
- Assembly gates (phase_completion flags unchanged)
- All other skill interfaces

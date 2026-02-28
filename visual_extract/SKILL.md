---
name: anime-channel-visual
description: Visual generation agent for the anime channel. Generates all anime-style images and video clips. Reads tools registry for provider selection, implements correct xSkill async polling, uses image references for character consistency, and verifies every file before marking complete.
---

# Anime Channel Visual Agent

## Read These First
1. `anime-channel-tools/SKILL.md` — provider registry, language routing, quality suffixes, cam-ref library
2. `workspace/in-progress/state.json` — current pipeline state, character cards, style anchor

Check phase from state.json:
- `"scene_1_anchor_test"` → generate scene 1 only, return for owner approval
- `"batch_N"` → generate the specified scene batch (max 3 per session)
- `"cam_ref_bootstrap"` → generate the camera reference library (one-time setup, see below)

---

## PROVIDER ABSTRACTION LAYER

This skill NEVER hardcodes a provider. All provider decisions come from the tools registry.

The tools registry defines capabilities. This skill asks for a capability, not a model name:
- `IMAGE_PRIMARY` — best quality image generation (Seedream — Chinese, zh prompts)
- `IMAGE_FAST` — fast/cheap image generation (Flux — western, en prompts)
- `VIDEO_STANDARD` — standard video (Seedance — Chinese, zh prompts)
- `VIDEO_CINEMATIC` — western provider for sensitive content (Runway — en prompts)
- `VIDEO_FAST` — cheap video for establishing shots (Kling — Chinese, zh prompts)

**This skill never needs to change when providers change.** Only the tools registry changes.

---

## PROMPT LANGUAGE ROUTING

### Step 1 — Check prompt_language from tools registry

Read `tools_registry.[CAPABILITY].prompt_language`:
- `"zh"` → build prompt in English, translate to Chinese before submitting, store both
- `"en"` → build prompt in English, submit English, done

### Step 2 — Build base prompt in English always

Scriptwriter always writes in English. Character cards in English. Style anchor in English.
All source material is English — it's the canonical layer you can read and review.

### Step 3 — Translate when needed (zh providers only)

When `prompt_language = "zh"`, call Claude (yourself) to translate before submitting:

```
Translate this anime scene description to Chinese for AI video generation.
Keep all technical cinematic terms accurate.
Keep camera movements, lighting terms, and style references precise.
Use natural, colloquial Chinese — describe it like you'd tell a friend what you want to see,
not like a film school brief. Colloquial language performs better on models trained on 
Chinese social video data.
Do NOT add or remove any creative details.

[English prompt here]
```

### Step 4 — Store both in state.json

```json
"generation_jobs": {
  "scene_3": {
    "prompt_en": "Close-up of samurai drawing sword, golden hour light, dust particles...",
    "prompt_submitted": "武士拔剑特写，黄金时段光线，尘埃颗粒飞散...",
    "prompt_language": "zh",
    "provider": "xskill:seedance_2.0_fast",
    "status": "submitted"
  }
}
```

**Why dual storage:** You review `prompt_en` — always readable. `prompt_submitted` is what actually went to the API. If a generation fails, retry from `prompt_en` and re-translate for any fallback provider. If the fallback uses a different language, the retry logic re-runs translation automatically.

---

## 8-DIMENSION PROMPT FORMULA

**Order matters.** Chinese model training data follows this pattern. Put elements in this sequence:

```
[1. STYLE ANCHOR] → [2. SUBJECT] → [3. ACTION] → [4. ENVIRONMENT/LIGHTING] → [5. CAMERA] → [6. SOUND] → [7. CONSTRAINTS] → [8. QUALITY SUFFIX]
```

### Dimension guide

**1. Style Anchor** — pulled from state.json style_anchor.description
```
"Anime cinematic style, dark seinen palette, Studio Trigger aesthetic"
"Ghibli-influenced, watercolor backgrounds, warm soft lighting"
"Wuxia ink-wash aesthetic, high contrast, bold silhouettes"
```

**2. Subject** — who or what, with specific physical detail
```
"A young samurai, black hair, red haori, worn katana at his hip"
"An elderly scholar with wire-rimmed glasses, ink-stained hands"
```
Include character card `prompt_fragment` verbatim if character appears.

**3. Action** — what is happening. Use slow, deliberate action words.
```
Good: "slowly draws his sword, dust settling around his feet"
Bad: "fighting intensely" ← too vague, model invents its own version
Good: "the blade catches the last light as it clears the scabbard"
```
One action per clip. "One scene, one thing" — split if you need more beats.

**4. Environment + Lighting** — where and how it's lit
```
"on a cliff edge at golden hour, sun low behind mountains, long shadows"
"inside a dimly lit archive, single lamp, dust motes in the beam"
"a snow-covered village square at dusk, fires burning in barrels"
```
Specify light source and direction. Model physics activate when you describe force:
```
Good: "waves hit rocks and explode upward, spray catches backlight"
Bad: "waves crash" ← model guesses
```

**5. Camera** — how we see it
```
"close-up, slow push in toward face"
"wide shot, low angle looking up"
"refer to @video1 camera movement"  ← use cam-ref library when possible
```
For complex camera moves, reference a cam-ref video instead of describing in text.
The model reads movement from video far more accurately than from text description.

**6. Sound** (video only — this is Seedance's audio generation layer)
```
"sound of steel on steel, distant thunder, no music"
"wind through bamboo, cicadas, footsteps on gravel"
"crowd noise fading to silence, single heartbeat"
```
Be specific: "the metallic clink of a coin" generates a sharper, more synced sound
than "coin sound." Specify reverb, distance, texture.

**7. Constraints** — what NOT to do
```
"no subtitles, no BGM, no watermark, no text overlay"
← these are ALWAYS included, pulled from tools registry quality suffix constants
```
Always explicitly state these. Seedance adds BGM/subtitles to ~80% of generations
by default. If you don't exclude them here, Assembly has to strip AI audio from
every clip before compositing your actual audio — avoidable cost and complexity.

**8. Quality Suffix** — appended automatically from tools registry constants
```
zh: 画面稳定无抖动，高清，面部清晰不变形，五官自然，细节丰富，电影质感，不要BGM，不要字幕
en: stable shot no camera shake, highly detailed, face clear and consistent, 
    cinematic texture, masterpiece, best quality, no text, no watermark, 
    no background music, no subtitles
```

### Full assembled example (zh)

English build:
```
[Anime cinematic dark seinen style], [young samurai, black hair, red haori, worn katana], 
[slowly draws sword, dust settling around feet], [cliff edge at golden hour, sun low behind mountains], 
[close-up, slow push in], [steel whisper, wind, no music], [no subtitles no BGM]
```

After translation + quality suffix appended:
```
动漫电影风格，黑暗青年漫风格，年轻武士，黑发，红色羽织，腰间旧刀，缓缓拔剑，
尘埃在脚边沉落，悬崖边，黄金时段，夕阳在山后低垂，近景，缓慢推镜，
钢铁低语声，风声，无音乐，画面稳定无抖动，高清，面部清晰不变形，五官自然，
细节丰富，电影质感，不要BGM，不要字幕
```

---

## ONE SCENE, ONE THING

**The most common mistake:** stuffing multiple beats into one clip.

Each video generation call handles exactly ONE thing:
- One action
- One emotional beat
- One camera angle

If your scene needs more: split into two clips and stitch in Assembly.

**Wrong:**
```
"Samurai draws sword, fights three guards, defeats them all, sheathes sword and walks away"
```

**Right — 3 separate clips:**
```
Clip A: "Samurai's hand rests on sword hilt, eyes narrow, guards notice him"
Clip B: "Guard lunges — samurai sidesteps, blade catches light"  
Clip C: "Samurai sheathes sword slowly, three guards on ground behind him"
```

Scriptwriter should have split these at script stage. If they didn't, Visual agent splits them.
Flag it in state.json and let Director know the scene count increased.

---

## CONTENT ROUTING BY STORY TYPE

Before generating any video, check `state.json.story_tags[]`.

Chinese models (Seedance, Kling) have strict filters on:
- Political figures, protests, democracy themes
- Historical atrocities and war crimes
- LGBTQ+ representation in certain contexts
- Celebrity face references (tightened Feb 2026)

```
story_tags contain: war / battle / military / historical_violence / political_conflict
                    / protest / revolution / war_crime / atrocity
→ VIDEO: Route to tools_registry.VIDEO_CINEMATIC (Runway — western, en prompts)
→ IMAGES: IMAGE_PRIMARY still fine (Seedream/Flux handle historical imagery)
→ Log: state.json.content_routing_notes = "Routed to western provider: [reason]"

story_tags contain: supernatural / sci-fi / romance / adventure / mythology
                    / slice_of_life / heist / detective / martial_arts
→ VIDEO: tools_registry.VIDEO_STANDARD (Seedance, zh prompts)

story_tags contain: landscape / atmosphere / establishing / abstract / nature
→ VIDEO: tools_registry.VIDEO_FAST (Kling, zh prompts — cheaper)
```

**On content policy failure at generation time:**
1. Log exact error to state.json pipeline_log
2. Rewrite prompt — remove flagged terms, use indirect/historical framing
3. Retry same provider once
4. If still fails → switch to VIDEO_CINEMATIC automatically
5. If still fails → degrade to static image (log as content_policy_flag, not failure)

---

## 9-GRID STORYBOARD TRICK (optional, complex scenes)

For scenes with multiple camera angles or narrative beats in rapid succession — like a
fight choreography or a flashback montage — use the 9-grid technique:

1. **Generate a 9-panel storyboard image** using IMAGE_FAST:
   - Create a prompt describing 9 sequential frames of the scene
   - The image model generates them as a single 3×3 grid image
   - Each panel has a simple action moment, left-to-right, top-to-bottom order

2. **Pass the grid image to Seedance as a reference:**
   ```
   prompt: "@图片1 按照顺序演绎分镜内容，保持人物一致，动作流畅连贯，音效恰到好处"
   ("@image1 perform the storyboard panels in sequence, keep characters consistent,
     actions fluid and continuous, sound effects well-timed")
   image_files: [grid_image_url]
   functionMode: "omni_reference"
   duration: 10-15  ← use longer duration for multi-beat sequences
   ```

3. **What Seedance does:** Reads each panel as a sequential shot. Auto-generates
   transitions, scene changes, matching character appearance. Can encode a full
   fight sequence into a single generation call.

**When to use this:**
- Fight choreography with 4+ beats
- Flashback montage sequences
- Any scene where splitting into individual clips would cost 3× more

**When NOT to use this:**
- Simple scenes (use standard generation — faster and more controllable)
- Scenes with characters requiring strict consistency (direct image reference is more reliable)

---

## CAM-REF LIBRARY BOOTSTRAP

One-time setup when `assets/cam-refs/` doesn't exist.
Called by Director as first pipeline step on new installation.

Generate each reference clip using VIDEO_FAST (cheap — these are reference-only):

```
cam_push_in_slow: "camera slowly pushes toward center of empty frame, neutral grey background, 3 seconds, no subject"
cam_pull_out_reveal: "camera pulls back from close to wide shot revealing empty landscape, 3 seconds"
cam_orbit_left: "camera orbits 180 degrees around a vertical object, neutral background, 4 seconds"
cam_crane_rise: "camera rises from ground level to sky, simple outdoor setting, 3 seconds"
cam_handheld_tense: "handheld camera, slight shake, close environment, 3 seconds"
cam_whip_pan: "fast whip pan left to right, motion blur, 2 seconds"
cam_first_person: "first-person POV walking forward through a corridor, 4 seconds"
```

Save each to `assets/cam-refs/`. Upload each to xSkill (`upload_image` tool also handles video).
Save stable URLs to `state.json.cam_ref_library`:

```json
"cam_ref_library": {
  "push_in_slow": "https://xskill.ai/stable/cam_push_in_slow.mp4",
  "pull_out_reveal": "https://xskill.ai/stable/cam_pull_out_reveal.mp4",
  ...
}
```

Total cost: ~140 credits one-time. Reused across every video indefinitely.

---

## CREDIT CHECK

Always before starting any batch:
```
Call: get_balance
If credits < 100 → warn owner, pause, do not proceed
Log balance to state.json.last_credit_check
```

---

## SCENE 1 ANCHOR TEST

When called with `"scene_1_anchor_test"`:

### 1. Route provider and language
```
Check story_tags → determine correct capability (VIDEO_STANDARD vs VIDEO_CINEMATIC etc.)
Read tools_registry.[capability].prompt_language → zh or en
```

### 2. Build prompt using 8-dimension formula
```
style_prefix = state.json style_anchor.description
scene = state.json script.scenes[0]

Build in English following 8-dimension order.
If prompt_language = zh → translate.
Append quality suffix from tools registry.
```

### 3. Generate with sync_generate_image (anchor is always an image first)
```
Tool: sync_generate_image
model: [tools_registry.IMAGE_PRIMARY]
prompt: [built above — in correct language]
```

### 4. Download and verify
```
image_url = response.url
Download to: assets/images/scene_1_anchor.png

Verify:
  stat assets/images/scene_1_anchor.png → must exist
  file size > 10KB → if smaller, corrupt — retry
```

### 5. Upload for stable reference URL
```
Tool: upload_image
file: assets/images/scene_1_anchor.png

Returns: stable_url
Save to state.json: style_anchor.xskill_reference_url = stable_url
```

### 6. Extract character card
From scene description, identify named/recurring characters.
Write to state.json:
```json
"character_cards": {
  "protagonist": {
    "description_en": "exact physical description in English",
    "description_zh": "translated version for zh prompts",
    "xskill_reference_url": "[stable_url from step 5]",
    "prompt_fragment_en": "reusable English text: appearance, clothing, physical features",
    "prompt_fragment_zh": "Chinese version of same fragment"
  }
}
```

### 7. Save and report
```
state.json:
  style_anchor.prompt_en = [English prompt before translation]
  style_anchor.prompt_submitted = [exact prompt sent to API]
  style_anchor.prompt_language = [zh or en]
  style_anchor.approved_scene_1_path = "assets/images/scene_1_anchor.png"
  generation_jobs.scene_1.status = "complete"
  generation_jobs.scene_1.file_path = "assets/images/scene_1_anchor.png"
```

Report to Director. Director shows owner. Wait for approval before proceeding.

---

## BATCH GENERATION

When called with `"batch_N"`:

Read state.json to find scenes in this batch. Skip any with `status: "complete"`.

### Step 1 — Decide: image or video?

**Use image (sync_generate_image) for:**
- Establishing shots, wide shots
- Dialogue / conversation scenes
- Aftermath / reflection / contemplation
- Any scene where "a moment frozen in time" works narratively
- Any scene whose story_tag triggered content routing to avoid video filters

**Use video (submit_task) for MAX 3 scenes per video:**
- Inciting incident
- Climax / battle / confrontation
- Resolution / final moment
- Scenes explicitly marked `force_video: true` in script

When in doubt → image. Video is 8–30× more expensive and takes 10 minutes.

### Step 2 — Apply "one scene, one thing" check
Read the scene visual_description. If it contains multiple actions or beats, split before generating.
Add new sub-scenes to state.json, notify Director of count change.

### Step 3 — Build prompt (8-dimension formula)

```
[1] style_anchor.description
[2] character_card.prompt_fragment (if character appears)
[3] scene.visual_description — main action (one thing only)
[4] lighting + environment from scene
[5] camera movement — reference cam_ref_library URL if applicable
[6] sound description (video only)
[7] "no BGM, no subtitles" (always)
[8] quality suffix from tools registry (language-matched)
```

Translate to zh if `prompt_language = "zh"`.
Store both `prompt_en` and `prompt_submitted` in state.json generation_jobs.

### Step 4 — Submit

**For images:**
```
Tool: sync_generate_image
model: [tools_registry.IMAGE_PRIMARY or IMAGE_FAST]
prompt: [built above — in correct language with suffix]

Result comes back immediately.
Extract URL. Download to: assets/images/scene_N.png
```

**For video (omni_reference — standard):**
```
Tool: submit_task
Outer model: "st-ai/super-seed2"

params:
  prompt: [built above — in correct language with suffix]
  functionMode: "omni_reference"
  image_files: [character_card.xskill_reference_url]   ← for character consistency
              OR [style_anchor.xskill_reference_url]   ← for atmosphere-only scenes
  video_files: [cam_ref_library.push_in_slow]          ← add if using cam reference
  ratio: "16:9"
  duration: 8      ← INTEGER, 4–15
  model: "seedance_2.0_fast"
```

**For video (first_last_frames — transitions):**
```
Tool: submit_task
Outer model: "st-ai/super-seed2"

params:
  prompt: [transition description in correct language]
  functionMode: "first_last_frames"
  filePaths: ["[start_frame_url]", "[end_frame_url]"]
  ratio: "16:9"
  duration: 5
  model: "seedance_2.0_fast"
```

**Seedance limits (hard):**
- Max 9 image_files + 3 video_files + 3 audio_files per call
- Reference video/audio total duration ≤ 15 seconds
- No real human face uploads — use generated character images only
- duration must be integer 4–15

**Immediately after submit_task returns:**
```
Write to state.json:
  generation_jobs.scene_N.job_id = task_id
  generation_jobs.scene_N.status = "submitted"
  generation_jobs.scene_N.submitted_at = [timestamp]
  generation_jobs.scene_N.prompt_en = [english prompt]
  generation_jobs.scene_N.prompt_submitted = [actual submitted prompt]
  generation_jobs.scene_N.prompt_language = [zh or en]
  generation_jobs.scene_N.provider = [capability + model used]

DO NOT call get_task. DO NOT wait. Move to next scene immediately.
```

After ALL jobs submitted → batch session ENDS. Director's polling loop handles rest.

### Step 5 — Verify images (sync only)
```
stat assets/images/scene_N.png → must exist
file size check: > 10KB (if smaller, corrupt — retry, max 3 attempts)
On success:
  state.json generation_jobs.scene_N.status = "complete"
  state.json generation_jobs.scene_N.file_path = "assets/images/scene_N.png"
```

---

## POLLING AGENT

Director spawns this periodically while video jobs are pending.

```
Read state.json generation_jobs
For each job with status = "submitted":

  Call: get_task
  params: { task_id: [job.job_id] }
  
  Response format:
  {
    "code": 200,
    "data": {
      "status": "pending" | "processing" | "completed" | "failed",
      "result": {
        "output": {
          "images": ["https://video-output-url.mp4"]
        }
      }
    }
  }

  status = "pending" | "processing":
    Log elapsed time. Exit poll. Wait for next cycle.

  Polling cadence:
    Images (Seedream): first check 30s, then every 30s, timeout 180s
    Videos (Seedance): first check 60s, then every 90s, timeout 600s
    After timeout → log warning, keep polling — don't abort. Queue loads vary.

  status = "completed":
    video_url = data.result.output.images[0]
    Download: curl -L "[video_url]" -o assets/videos/scene_N.mp4
    Verify: file size > 100KB (if smaller, download failed — retry once)
    
    Update state.json:
      generation_jobs.scene_N.status = "complete"
      generation_jobs.scene_N.file_path = "assets/videos/scene_N.mp4"
      generation_jobs.scene_N.completed_at = [timestamp]
      
  status = "failed":
    Log full error to state.json pipeline_log
    state.json generation_jobs.scene_N.retry_count += 1
    
    Retry logic:
      retry 1 → rewrite prompt_en (simplify, remove flagged terms), re-translate if zh, resubmit
      retry 2 → switch to fallback provider from tools_registry for this capability
      retry 3 → degrade: generate static image with sync_generate_image
      retry 4 → mark "skipped", log, continue — never crash pipeline over one scene

After all jobs polled:
  All complete/degraded/skipped → state.json phase_completion.visual = true
  Otherwise → report "X/N scenes complete, Y pending" to Director
```

---

## THUMBNAIL GENERATION

After all scenes complete, generate 2 thumbnail options.
Use `IMAGE_PRIMARY` — highest quality, this is the face of the video.

Build in English, translate to zh if IMAGE_PRIMARY is a zh provider.

```
Prompt A (character close-up):
[style_anchor + character], extreme contrast, vivid saturated colors,
[emotional expression from climax], painterly quality, 16:9
+ quality suffix

Prompt B (epic scene wide):
[style_anchor], [key dramatic wide scene from climax],
epic scale, golden ratio composition, atmospheric depth, dynamic lighting, 16:9
+ quality suffix
```

Download both. Verify both (> 10KB).
Save: assets/images/thumbnail_A.png, thumbnail_B.png
Update state.json with both paths and both prompt_en / prompt_submitted values.

---

## ERROR HANDLING

All errors follow resilience.skill retry matrix.
Every error logged to state.json pipeline_log with exact error text.
Never swallow errors silently — degraded scenes logged explicitly.

**Content policy failures:**
```json
{
  "scene": "N",
  "provider": "xskill:seedance_2.0_fast",
  "prompt_en": "first 100 chars of English prompt",
  "prompt_submitted": "first 100 chars of submitted prompt",
  "error": "exact error message from API",
  "action_taken": "switched_provider | prompt_rewrite | degraded_to_image"
}
```

**Video → image degradation:**
```json
{
  "scene": "N",
  "reason": "video generation failed after 3 retries",
  "degraded_to": "static_image",
  "original_error": "exact error"
}
```

Owner sees degradation summary at preview time — not mid-run. Pipeline continues.

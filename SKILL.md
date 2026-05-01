---
name: autoshorts
description: "Daily pipeline that picks one long video from a folder, transcribes it with Whisper, uses Gemini 3 Flash multimodal to find every viral short-form moment, cuts each candidate with FFmpeg, adds a hook-text overlay, presents the candidates to the user for approval, and publishes the approved clips to TikTok / Instagram Reels / YouTube Shorts via the Upload-Post API. Use when the user wants to create shorts/reels/clips from longer videos, mentions autoshorts, viral clips, or content repurposing, or asks for the daily clip batch."
version: "2.0.0"
---

# AutoShorts — Daily Viral Clip Pipeline

Pipeline tooling lives at `~/Documents/skill-autoshorts/`. Each day this skill picks ONE long video from `INPUT_FOLDER`, extracts every viable short-form clip (Gemini 3 Flash decides), shows them to the user for approval, and publishes the approved ones via Upload-Post.

## Setup (only if not yet configured)

### 1. Python environment
```bash
cd ~/Documents/skill-autoshorts && python3 -m venv venv && source venv/bin/activate && pip install -r requirements.txt
```

### 2. FFmpeg
Required system binary. Verify with `ffmpeg -version`. Install with `brew install ffmpeg` if missing.

### 3. `.env`
File lives at `~/Documents/skill-autoshorts/.env`. Required keys:

```
UPLOAD_POST_API_KEY=...
UPLOAD_POST_PROFILE=...
GEMINI_API_KEY=...
INPUT_FOLDER=/abs/path/to/long/videos
OUTPUT_FOLDER=/abs/path/to/clip/output
WHISPER_MODEL=medium
TIMEZONE=Europe/Madrid
```

If a required key is missing, ask the user for it before continuing.

### 4. Upload-Post account
- Sign up at https://upload-post.com → dashboard at https://app.upload-post.com.
- Connect TikTok, Instagram (Business/Creator account linked to a Facebook Page), and YouTube via OAuth in the dashboard.
- In **Manage Users**, create a profile — its name is `UPLOAD_POST_PROFILE` (NOT the social handle).
- Generate an API key in **Settings**.
- Verify: `curl -H "Authorization: Apikey $UPLOAD_POST_API_KEY" https://api.upload-post.com/api/uploadposts/me`.

## Orchestration model

This skill is invoked daily by the **openclaw** harness, which also handles the messaging bridge (Telegram, WhatsApp, or whatever channel openclaw is configured with). The skill itself does NOT talk to Telegram or any messenger directly — it just runs the pipeline and presents the candidates as text + absolute file paths. openclaw forwards your output to the user's phone, captures the user's reply, and feeds it back into the conversation.

Concretely: at Step 5 you print the candidates table and ask which IDs to publish; openclaw delivers that table plus the clip files via the user's chosen channel; the user replies on their phone (e.g., "1, 3, 5"); openclaw injects that reply back; you continue with Steps 6–8. Same pattern for any other "ask the user" point in the workflow (metadata review, dry-run confirmation, etc.).

If the skill is invoked outside openclaw (e.g., user runs `/autoshorts` directly in Claude Code), the same prompts work — they just appear in the terminal instead of on the phone.

## Daily workflow

This skill is meant to run as a **daily infinite loop**. Every run picks ONE video that has not been processed yet and walks it through the pipeline. The state file at `state/processed.json` is the source of truth — once a video's sha256 is in there, it is never picked again. If the user adds a new video tomorrow, it gets priority over older unprocessed ones (sort is mtime DESC), so fresh material always jumps the queue. Old unprocessed videos still get picked eventually as the queue drains.

### Step 0 — Preflight (run on every invocation, do not skip)

Before doing any work, check that the environment is ready and ask the user for whatever is missing:

1. **venv** — does `~/Documents/skill-autoshorts/venv/bin/python` exist? If not, run setup step 1 from the Setup section. (You can do this without asking — it's mechanical.)
2. **`ffmpeg`** in `PATH` — if missing, ask the user to `brew install ffmpeg` (do not install yourself; system-wide installs deserve confirmation).
3. **`.env` file** — check that every required key is set and non-empty:
   - `GEMINI_API_KEY` → if missing, ask: *"Falta la API key de Gemini. Pégamela (la generas en https://aistudio.google.com/apikey)."*
   - `UPLOAD_POST_API_KEY` and `UPLOAD_POST_PROFILE` → if missing, ask: *"Necesito la API key de Upload-Post y el nombre del profile (Manage Users en https://app.upload-post.com)."*
   - `INPUT_FOLDER` and `OUTPUT_FOLDER` → if missing, default to `~/Documents/skill-autoshorts/input` and `~/Documents/skill-autoshorts/output` and write them to `.env`.
   - `WHISPER_MODEL` → default `medium`. `TIMEZONE` → default `Europe/Madrid`.
4. **Upload-Post platform health** — call `GET /api/uploadposts/users` and read `reauth_required` for each platform on the configured profile. If any platform requires reauth, surface it now so the user knows to either reauth (https://app.upload-post.com) or drop it from `--platforms` later.

If the user provides an API key in the conversation, write it to `.env` immediately, never echo it back, and **warn that the key is now in conversation logs and they should rotate it after testing.**

**How videos arrive into `INPUT_FOLDER`** is the harness's job, not the skill's. The canonical flow: the user forwards a video to openclaw / Hermes / their agent in chat (Telegram / WhatsApp / etc.), the harness downloads it and saves it to `INPUT_FOLDER`. The skill itself only operates on files that are already there. If the user passes a video path that is NOT inside `INPUT_FOLDER` (e.g. `/autoshorts /Users/foo/Downloads/podcast.mp4`), copy it in first (use `cp`, do not move — the original stays put). Otherwise `pick` will not find it.

### Step 1 — Pick the video

```bash
python autoshorts.py pick
```

Returns JSON with the next unprocessed video (newest mtime first, drops anything whose sha256 is already in `state/processed.json`). If the user just dropped a new file in the folder it will be picked first. If `remaining_unprocessed: 0` and the command exits with "all videos already processed", **stop and tell the user**:

> *No hay videos nuevos en `INPUT_FOLDER`. Suelta uno y vuelve a invocar la skill.*

Do NOT try to bypass the state file or reprocess anything — that's a hard rule. If the user explicitly says "reprocess video X", remove that entry from `state/processed.json` first, then run pick.

### Step 2 — Transcribe

```bash
python autoshorts.py transcribe "<VIDEO_PATH>"
```

Writes `output/<video_slug>/transcript.json` with sentence segments and per-word timestamps. Whisper auto-detects language. Default model is `medium`.

### Step 3 — Analyze with Gemini 3 Flash

```bash
python autoshorts.py analyze "<VIDEO_PATH>"
```

Uploads the video to Gemini Files API and asks `gemini-3-flash-preview` to return EVERY viable short-form moment (20–60s each), with timestamps snapped to word boundaries from the transcript. Output: `output/<video_slug>/clips.json`. Read this file to get the candidate list.

### Step 4 — Cut every candidate and add hook overlay

For each clip in `clips.json`, run two commands:

```bash
python autoshorts.py extract "<VIDEO_PATH>" \
    --start <START> --end <END> \
    --output "output/<slug>/clip_<ID>.mp4"

python autoshorts.py hook "output/<slug>/clip_<ID>.mp4" \
    --text "<HOOK_TEXT>" --duration 3 \
    --output "output/<slug>/clip_<ID>_final.mp4"
```

The hook is rendered TikTok/Instagram-style: each line of text gets its own black pill (78% opacity, rounded corners) behind it, with white Impact text + black stroke on top. The pill keeps the hook legible on any background — pure white, pure black, busy screenshares — without needing to inspect the underlying frame. Positioned at the top of the frame for the first 3 seconds. Hook text comes from `clips.json` (Gemini wrote it in the video's language).

Cut and hook ALL candidates upfront — the user will review the actual final files visually, not metadata.

### Step 4.5 — Visual QA of the hook (you do this yourself, no Gemini call)

You are multimodal. **Use that.** Before showing the candidates to the user, verify the hook overlay actually renders cleanly on each clip.

For every `clip_<ID>_final.mp4`:

```bash
python autoshorts.py preview output/<slug>/clip_<ID>_final.mp4
```

This extracts a single frame at t=1.0s (mid-hook) to `preview_clip_<ID>_final.png` next to the clip. Open it with the **Read** tool — Claude / openclaw both view PNGs directly. No Gemini call needed; the agent running the skill IS the multimodal reviewer.

For each preview, evaluate:

1. Is the hook text fully visible? Any letter clipped at the left/right/top edges?
2. Is the pill background extending past the safe area (more than ~5% from any edge)?
3. Does the hook cover the speaker's face or other critical content?
4. Are accent marks / special characters (`á é í ó ú ñ ¿ ¡`) rendering correctly?
5. Is the hook overlapping with the burned-in subtitle? (Subtitles at the bottom are expected — only flag if they collide with the hook itself, which lives at the top.)
6. Any rendering glitch: garbled text, missing pill, transparency issue?

**Add a "QA" column to the Step 5 table** with one of:
- `✅` — clean
- `⚠️ <issue>` — flag the specific problem (e.g. `⚠️ último carácter recortado`, `⚠️ pill desbordado a la derecha`)

**Do NOT silently drop flagged clips** — show them to the user with the warning so they can decide. The QA pass is advisory: a "⚠️" is a hint, not a veto. If multiple clips fail in the same way (e.g. the hook is consistently overflowing), that's a signal to suggest the user shorten the hook style going forward.

### Step 5 — Present to the user

Show a markdown table:

| ID | Duration | Hook | Score | QA | Reason | File |
|----|----------|------|-------|----|--------|------|
| 1  | 38s      | "..."| 9     | ✅ | ...    | output/<slug>/clip_1_final.mp4 |
| 2  | 27s      | "..."| 7     | ⚠️ acento "ó" recortado | ... | output/<slug>/clip_2_final.mp4 |
| …  | …        | …    | …     | …  | …      | … |

**Always include the absolute file paths in the table** — openclaw uses them to attach the actual clip videos when it forwards the message to the user's messenger (Telegram / WhatsApp / etc.). Without absolute paths the user sees only metadata and cannot review the clips visually. Then ask:

> **Which clip IDs do you want to publish? (e.g. `1, 3, 5`, or `none`.)**

Wait for the user's reply (it will arrive via openclaw from the user's phone).

**If the user replies `none`** (rejects all candidates), skip directly to Step 8 and `mark-processed` with `--clips-published 0`. This consumes the video so tomorrow's run picks the next one — otherwise the same rejected candidates would surface again. If the user wants to retry the same video later, they can manually remove its entry from `state/processed.json`.

### Step 6 — Generate platform metadata for approved clips

For every approved ID, generate platform-specific copy. **This is YOUR job as Claude** — write it directly, do not call a tool. Match the language of the video.

- **TikTok** (`tiktok_title`, max 90 chars): punchy hook, 1–2 emojis, hashtag mix at end of the title. Sweet spot ~70–85 chars.
- **Instagram Reels** (`instagram_title`, up to 2200 chars): long-form storytelling — first line is the hook, then 2-4 short paragraphs (use `\n\n`), CTA ("Guarda esto", "Etiqueta a alguien…", "Comenta X para…"), then 20-30 hashtags mixing sizes (large/medium/niche). Sweet spot 500–800 chars total.
- **YouTube Shorts** (`youtube_title`, max 100 chars but **keep ~40-60 chars** so it doesn't truncate on mobile): SEO-friendly with keywords. Description focuses on searchability, 3–5 hashtags max.
- A general `title` and `description` for any platform that doesn't have its own override.

**Length contract (verify before publishing):** YouTube title is the most constrained — write it shortest and most direct. TikTok and Instagram can breathe — TikTok up to ~85 chars in `tiktok_title`, Instagram captions are long-form by design.

Show the generated copy back to the user and confirm before publishing.

### Step 7 — Schedule publishing

Schedule one approved clip per day starting tomorrow at **10:00** in `TIMEZONE` (default `Europe/Madrid`). Each next clip += 1 day.

**Before publishing**, verify connected platforms and reauth status:

```bash
curl -s -H "Authorization: Apikey $UPLOAD_POST_API_KEY" \
    https://api.upload-post.com/api/uploadposts/users | python -m json.tool
```

If any platform shows `"reauth_required": true`, warn the user — that platform's upload will fail. Either drop that platform from `--platforms` or pause and let the user reauthorize in https://app.upload-post.com.

For each approved clip:

```bash
python autoshorts.py publish "output/<slug>/clip_<ID>_final.mp4" \
    --platforms tiktok,instagram,youtube \
    --title "<GENERAL>" \
    --description "<DESCRIPTION>" \
    --tiktok-title "<TIKTOK_TITLE>" \
    --instagram-title "<INSTAGRAM_CAPTION>" \
    --youtube-title "<YOUTUBE_TITLE>" \
    --schedule "<ISO_DATE>" \
    --timezone "Europe/Madrid" \
    --tiktok-mode draft \
    --clip-id <ID> \
    --hook-text "<HOOK_TEXT>" \
    --viral-score <GEMINI_SCORE> \
    --reason "<GEMINI_REASON>" \
    --video-source "<SOURCE_VIDEO_FILENAME>"
```

**The `--clip-id`, `--hook-text`, `--viral-score`, `--reason`, `--video-source` flags are not optional in practice** — they feed the learning loop. Without them, `learn` cannot correlate engagement metrics back to which hook patterns worked. The values come straight from `clips.json` (the Gemini output) and the source video filename.

**TikTok mode**: `--tiktok-mode draft` (default) sends to the TikTok inbox via `post_mode=MEDIA_UPLOAD` so the user can finish editing in-app before publishing. Use `--tiktok-mode direct` (`DIRECT_POST`) only when the user explicitly wants immediate publishing.

**Always run with `--dry-run` first** and show the user the exact request payloads. Only execute the real publish after explicit "go".

### Step 8 — Mark video as processed

```bash
python autoshorts.py mark-processed "<VIDEO_PATH>" \
    --clips-generated <N_CANDIDATES> \
    --clips-published <N_APPROVED>
```

This appends the video's hash to `state/processed.json` so tomorrow's `pick` skips it. **Run this even if `--clips-published 0`** — a rejected video is still consumed. The only time you do NOT mark-processed is if the pipeline crashed mid-run (e.g., Gemini errored out before producing clips); in that case let the user retry the same video tomorrow.

### Step 8.5 — Reflect (optional, fast, qualitative)

After publishing, you can run a quick `reflect` to capture WHY the user approved the clips they approved (no engagement metrics needed — just the approved-vs-rejected signal):

```bash
python autoshorts.py reflect --window-days 30
```

This compares recent candidates (`learnings/candidate-history.jsonl`) against approvals (`learnings/post-history.jsonl`) and asks Gemini to extract qualitative patterns ("approves hooks with concrete numbers, rejects question-form hooks"). Output goes to `learnings/runs/reflect-YYYY-MM-DD-HHMM.md`.

These observations are NOT auto-promoted to HOT.md. They're notes for the user to review and curate. Run reflect occasionally — daily is overkill, weekly is fine.

### Step 9 — Final summary

Print:

| # | File | Duration | Hook | Schedule | Platforms |
|---|------|----------|------|----------|-----------|

…and the source video name with how many candidates were generated vs. published.

## Weekly learning loop (`learn`)

This skill **gets smarter over time**. Engagement data from past clips (views, likes, comments, shares, saves — fetched from Upload-Post analytics) is fed back into the clip-selection prompt for future runs.

### Cadence

Run `learn` **weekly**, not daily. Engagement metrics need time to mature; daily learn would chase noise.

```bash
python autoshorts.py learn
```

Defaults: 7-day soak (clips younger than this are excluded), 90-day max age (older are stale), composite score = 0.6·views + 0.4·engagement_rate, top/bottom 20% as winners/losers.

### What it does

1. Reads `learnings/post-history.jsonl` (every clip we published, with its hook + Gemini score + Gemini reason + source video).
2. For each clip in the soak window, calls `GET /api/uploadposts/post-analytics/{request_id}` — same `request_id` we got back at publish time.
3. Computes a composite score per clip and picks the top 20% (winners) and bottom 20% (losers).
4. Sends winners + losers + the existing `learnings/HOT.md` to Gemini Flash with a meta-prompt asking it to produce an updated HOT.md (≤80 lines) listing patterns supported by the new evidence.
5. Writes the new HOT.md (backing up the previous one as `HOT.YYYYMMDD-HHMMSS.md.bak`).
6. Writes a full audit to `learnings/runs/learn-YYYY-MM-DD.md` so the user can see exactly which clips were called winners/losers and how the learnings changed.

### How HOT.md feeds back

`cmd_analyze` automatically reads `learnings/HOT.md` (if it exists and is non-empty) and **prepends it to the Gemini prompt** as "PRIOR LEARNINGS — apply when selecting clips and writing hooks". Gemini then weighs those patterns when proposing clips and writing hooks for tomorrow's video. **You don't have to do anything to make this work** — it happens on every analyze call.

### When to run `learn`

- **Manually**, on demand: `python autoshorts.py learn`
- **Scheduled**, weekly via cron / openclaw: `0 9 * * 1 cd ~/Documents/skill-autoshorts && ./venv/bin/python autoshorts.py learn`
- **Skip** if `post-history.jsonl` has fewer than ~10 entries — the rule of "5 winners + 5 losers minimum" will short-circuit the run with a "not enough data" note.

### Things to NOT do

- Do not edit `HOT.md` by hand AND keep running `learn` — `learn` will overwrite your edits. If you want manual rules, put them in `learnings/insights/` (manual notes, not used by the pipeline).
- Do not delete `post-history.jsonl` or `metrics.jsonl` — they're append-only memory. Without them every `learn` starts from zero.
- Do not run `learn` more than ~once a week — Gemini will just churn the same patterns.

## Operating notes

- **Always confirm** before Step 4 (heavy ffmpeg work — do NOT skip, but confirm if Gemini returned > 15 candidates — could waste time), before Step 7 (publishing is irreversible once scheduled), and after Step 6 (metadata copy).
- If Gemini returns malformed JSON, the raw response is dumped to `output/<slug>/clips.raw.txt` — read it and re-prompt manually.
- Hook text comes from Gemini in the video's language. Do not translate.
- The Upload-Post free tier is **10 uploads/month** — one publish to 3 platforms counts as 3. Warn the user if scheduling would exceed the quota.
- All clip files are absolute paths under `OUTPUT_FOLDER/<video_slug>/`. Surface them clearly so the openclaw harness can attach them when forwarding to Telegram / WhatsApp / whatever messenger channel the user has configured.
- If `pick` says "all videos already processed", tell the user and stop — do not re-process. They need to drop a new video into `INPUT_FOLDER`.
- The state file at `state/processed.json` is the **only** memory between runs. Never edit it programmatically except via `mark-processed`. If the user asks to "reprocess video X", the right move is to ask them to confirm, then remove the matching entry from `state/processed.json` manually.
- The Whisper `medium` model (~1.5 GB) downloads on first transcribe call. Warn the user the first run will take longer — subsequent runs reuse the cached model.

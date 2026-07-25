---
name: vhscli
description: Use the `vhscli` CLI to analyze images/video/pdfs with a prompt, or generate images/videos/audio. Use when the user asks about local media, wants AI images/videos/audio, or mentions vhscli, vhs, seedream, seedance, seed-audio, nano-banana, or gpt-image.
---

# vhscli

`vhscli` is a command-line tool for multimodal AI: chat about
text/images/video/pdfs, or generate images, videos, and audio from prompts.
It's a thin client — auth, uploads, and model execution all happen
server-side, so users don't store any provider API keys locally.

Run `vhscli --help` or `vhscli <command> --help` to see current help — the CLI
is the source of truth.

## Invocation

Always run via `npx @getvhs/vhscli@latest` so you pick up the newest models,
flags, and fixes. Don't pin a version, and don't call a bare `vhscli` binary
even if one is on PATH — it may be stale.

```
npx @getvhs/vhscli@latest <command> ...
```

Throughout this doc, commands are written as `vhscli ...` for readability —
substitute `npx @getvhs/vhscli@latest ...` when running.

Requires Node.js ≥ 24.

## Top-level

```
vhscli [-v|--version] [-h|--help]
vhscli <command> [options] ...
```

- `-v`, `--version` — print version (only when no command is given)
- `-h`, `--help` — show help (works on root and every subcommand)

Commands:

- `login` — log in with google (opens browser; saves session to `~/.vhs/session.json`)
- `logout` — log out and delete local access tokens
- `whoami` — print the logged-in user's email
- `models` — list available models
- `generate <model> <prompt> -o <path>` — generate an image, video, or
  audio, wait, and save it (`-o` is required)
- `submit <model> <prompt> -o <path>` — submit the same task as `generate` but
  exit immediately, printing the task id; finish later with `resume <task_id>`
- `chat <prompt>` — chat with seed-2.0 / `a2:seed-2-pro` (text, image,
  video, or pdf input)
- `resume <task_id> -o <path>` — finish a submitted generation by its task id

The CLI stores nothing locally: a task lives in the VHS backend and the task
id is the only handle to it. There is no project, no database, no config —
see "How a task is tracked" below.

Every `generate`/`submit`/`resume` accepts `--json`: NDJSON events on stdout
(`submitted`/`progress`/`done`/`error`) with ordinary logs rerouted to
stderr — the machine-readable way to script generations (see "--json" below).

## Auth

Assume auth is already configured. If a command fails with an auth error, run
`vhscli login` to open a browser for Google OAuth. Do NOT run `vhscli login`
preemptively — it requires interactive browser login.

## Models

- **Chat / understand** (text / image / video / pdf): `seed-2.0`
  (`a2:seed-2-pro` under `vhscli chat`)
- **Generate images**: `seedream-5` (default), `seedream-5-pro`,
  `nano-banana-2`, `nano-banana-pro`, `gpt-image-2` — under `vhscli generate`
- **Generate video**: `seedance-2` — under `vhscli generate`
- **Generate audio**: `seed-audio-1` — under `vhscli generate`

## Prompt guides

Before you invoke `vhscli generate` (or do non-trivial understanding with
`vhscli chat`), **Read the matching prompt guide first** and shape the prompt
around it. The guides are concise, model-specific references distilled from
each provider's docs — formulas, what to lead with, what works, what fails.
Wording that's great for one model often underperforms on another, so don't
skip this.

| Model(s)                           | Guide file (Read before prompting) |
| ---------------------------------- | ---------------------------------- |
| `seed-2.0` (used by `vhscli chat`) | `prompt_guide/seed-2.txt`          |
| `seedream-5`, `seedream-5-pro`     | `prompt_guide/seedream.txt`        |
| `nano-banana-2`, `nano-banana-pro` | `prompt_guide/nano-banana.txt`     |
| `seedance-2`                       | `prompt_guide/seedance-2.txt`      |
| `gpt-image-2`                      | `prompt_guide/gpt-image-2.txt`     |

Trigger: any time the user asks for output from one of these models, Read its
guide before building the prompt. For trivial chat (plain text Q&A with no
media) you can skip `seed-2.txt`.

## Stdin prompts

Every command that takes a prompt also accepts `-` as the prompt, meaning
"read from stdin":

```
cat my_prompt.txt | vhscli generate nano-banana-pro - -o out.png
echo "what is this?" | vhscli chat - -i photo.jpg
```

---

## vhscli chat — chat about text, images, video, or pdfs

```
vhscli chat <prompt> [-i <image>...] [-f <pdf>...] [-v <video>] [--fps <n>]
```

Mode is picked from your flags:

- prompt only → text chat
- `-i` → ask about images (repeatable)
- `-f` → ask about pdf documents (repeatable)
- `-v` → ask about a single video

Options:

- `-i <path>` — image to ask about (repeat `-i` for more)
- `-f <path>` — pdf document to ask about (repeat `-f` for more)
- `-v <path>` — single video to ask about
- `--fps <n>` — frames/sec sampled from the video, 0.2–5 (default: 1)

One-shot — each call is independent, no memory of previous calls. Output goes
to stdout, nothing is saved to disk. Audio inside a video is not understood.

Examples:

```
vhscli chat "explain how to make sourdough in 5 steps"
vhscli chat "describe the scene. return json with objects, setting, mood." -i photo.jpg
vhscli chat "transcribe all visible text verbatim, preserving line breaks." -i receipt.jpg
vhscli chat "compare image 1 and image 2 in 3 bullets." -i a.jpg -i b.jpg
vhscli chat "summarize this paper in 5 bullets; include a page number per bullet." -f paper.pdf
vhscli chat "list key events with start_time and end_time in HH:mm:ss as json." -v clip.mp4 --fps 2
```

---

## vhscli generate seedream-5 — generate an image (default choice)

```
vhscli generate seedream-5 <prompt> -o <path> [-i <image>...] [--size <size>]
```

Options:

- `-o`, `--output <path>` — output file path, required (e.g. `out.jpg`)
- `-i <path>` — reference image, max 14 (repeat `-i` for more)
- `--size <size>` — `2K`, `3K`, or `WxH` like `1024x1536` (default: 2K)
  - WxH pixel count must be in [3,686,400, 10,404,496]
  - WxH aspect ratio must be in [1:16, 16:1]

Output format follows the `-o` extension (`.png`, `.jpg`/`.jpeg`, `.webp`); the
CLI converts if needed.

Examples:

```
vhscli generate seedream-5 "a red fox in a snowy forest" -o fox.jpg
vhscli generate seedream-5 "swap the outfit" -o out.png -i person.jpg -i outfit.jpg --size 3K
```

---

## vhscli generate seedream-5-pro — generate an image (Seedream 5.0, pro tier)

```
vhscli generate seedream-5-pro <prompt> -o <path> [-i <image>...] [--size <size>]
```

Options:

- `-o`, `--output <path>` — output file path, required (e.g. `out.png`)
- `-i <path>` — reference image, max 14 (repeat `-i` for more)
- `--size <size>` — `2K`, `3K`, or `WxH` like `1024x1536` (default: 2K)
  - WxH pixel count must be in [3,686,400, 10,404,496]
  - WxH aspect ratio must be in [1:16, 16:1]

Same prompt guide and same flags as `seedream-5`, but the pro tier: stronger
prompt adherence and finer detail, at roughly **2 minutes per image** instead
of seconds. Prefer plain `seedream-5` by default; reach for `seedream-5-pro`
when the user asks for the best quality, or when `seedream-5` keeps missing
details in a complex prompt. The provider renders png, so prefer a `.png` `-o`
(other extensions still work — the CLI converts).

Examples:

```
vhscli generate seedream-5-pro "a lone lighthouse on a cliff at dusk, long exposure, crashing surf" -o lighthouse.png
vhscli generate seedream-5-pro "add a flock of birds across the sky, keep the style" -i scene.png -o birds.png
```

Because it is slow, it pairs well with `vhscli submit` (below) when generating
several images: submit them all, then resume.

---

## vhscli generate nano-banana-2 — generate an image (Google)

```
vhscli generate nano-banana-2 <prompt> -o <path> [-i <image>...] [--size <size>]
```

Options:

- `-o`, `--output <path>` — output file path, required (e.g. `out.png`)
- `-i <path>` — reference image, max 14 (repeat `-i` for more)
- `--size <size>` — `1k`, `2k`, or `4k` (default: 1k), lowercase

`--size` picks a resolution tier, not exact dimensions — nano-banana takes no
`WxH`. The model chooses the aspect ratio, so describe the framing you want in
the prompt if you need a tall or wide composition.

Examples:

```
vhscli generate nano-banana-2 "remove the man from the photo, keep everything else" -i photo.jpg -o clean.png
vhscli generate nano-banana-2 "90s skateboarder poster, vertical composition" -o poster.png --size 2k
vhscli generate nano-banana-2 "a glossy candle in a bell jar on a marble counter, soft light" -o candle.png
```

---

## vhscli generate nano-banana-pro — generate an image (Google, premium)

```
vhscli generate nano-banana-pro <prompt> -o <path> [-i <image>...] [--size <size>]
```

Options:

- `-o`, `--output <path>` — output file path, required (e.g. `out.png`)
- `-i <path>` — reference image, max 14 (repeat `-i` for more)
- `--size <size>` — `1k`, `2k`, or `4k` (default: 1k), lowercase

Same resolution-tier sizing and model-chosen aspect ratio as nano-banana-2.
Higher-quality sibling — better text rendering and richer textures.

Examples:

```
vhscli generate nano-banana-pro "studio portrait, cinematic lighting, three-quarter framing" -o portrait.jpg --size 2k
vhscli generate nano-banana-pro "a sun-drenched minimalist living room with a 3d armchair from this sketch" -i sketch.jpg -o room.png
```

---

## vhscli generate gpt-image-2 — generate or edit an image (OpenAI)

```
vhscli generate gpt-image-2 <prompt> -o <path> [-i <image>...] [--size <size>]
```

Options:

- `-o`, `--output <path>` — output file path, required (e.g. `out.png`)
- `-i <path>` — reference image for edits (repeat `-i` for more)
- `--size <size>` — preset (`1024x1024`, `1536x1024`, `1024x1536`,
  `2048x2048`, `2048x1152`, `3840x2160`) or `WxH` (default: 1024x1024)
  - both sides must be multiples of 16, max edge 3840
  - total pixels in [655,360, 8,294,400]
  - aspect ratio in [1:3, 3:1]

Output format follows the `-o` extension (`.png`, `.jpg`/`.jpeg`, `.webp`); the
CLI converts if needed. Use png or webp when you need transparency.

Examples:

```
vhscli generate gpt-image-2 "a children's book drawing of a veterinarian examining a cat" -o vet.png
vhscli generate gpt-image-2 "replace the background with a starry night, keep the subject unchanged" -i photo.jpg -o night.png
vhscli generate gpt-image-2 "ultra-wide landscape of the swiss alps at golden hour" --size 3840x2160 -o alps.jpg
```

---

## vhscli generate seedance-2 — generate a video

```
vhscli generate seedance-2 <prompt> -o <path>
                           [--first-frame <image>] [--last-frame <image>]
                           [-i <image>...] [-v <video>...] [-a <audio>...]
                           [--ratio <r>] [--resolution <res>] [--duration <n>]
                           [--no-audio]
```

Mode is picked from your flags:

- prompt only → text-to-video
- `--first-frame` → animate from that frame (optionally `--last-frame` too)
- `-i` / `-v` / `-a` → use as references

Options:

- `-o`, `--output <path>` — output file path, required (`.mp4`, `.webm`, or
  `.mov`; prefer `.mp4`)
- `--first-frame <image>` — use as the first frame
- `--last-frame <image>` — use as the last frame (requires `--first-frame`)
- `-i <path>` — reference image, max 9 (repeat `-i`). conflicts with
  `--first-frame`
- `-v <path>` — reference video, max 3 (repeat `-v`). conflicts with
  `--first-frame`
- `-a <path>` — reference audio, max 3 (repeat `-a`). requires `-i` or `-v`,
  conflicts with `--first-frame`
- `--ratio <r>` — aspect ratio (default: 16:9). one of: `16:9`, `4:3`,
  `1:1`, `3:4`, `9:16`, `21:9`
- `--resolution <res>` — `480p`, `720p`, or `1080p` (default: 720p)
- `--duration <n>` — length in seconds, 4–15 (default: 5)
- `--audio` / `--no-audio` — toggle the audio track (default: `--audio`).
  pass `--no-audio` for a silent video

Defaults to 5s @ 720p, 16:9, with audio. Jobs run in the cloud and can take
minutes — the CLI polls automatically. If you don't want to block, use
`vhscli submit seedance-2 ...` (same flags) to detach immediately, then
`vhscli resume <task_id> -o cat.mp4` later (submit prints the id). A
`vhscli generate` interrupted mid-poll is finished the same way.

Examples:

```
# text-to-video
vhscli generate seedance-2 "a cat jumping off a couch" -o cat.mp4 --duration 6 --ratio 16:9

# animate a still image
vhscli generate seedance-2 "camera pans right" -o pan.mp4 --first-frame start.jpg

# with a first and last frame
vhscli generate seedance-2 "morph between these" -o morph.mp4 --first-frame a.jpg --last-frame b.jpg

# reference-based with audio
vhscli generate seedance-2 "lip sync the words" -o out.mp4 -i face.jpg -a voice.mp3
```

---

## vhscli generate seed-audio-1 — generate speech audio

```
vhscli generate seed-audio-1 <prompt> -o <path> [-i <audio>...]
```

Options:

- `-o`, `--output <path>` — output file path, required (must be `.mp3`)
- `-i <path>` — reference voice clip for cloning or blending, max 3
  (repeat `-i` for more). each clip should be ≤30s and ≤10MB

Output is always mp3.

Examples:

```
vhscli generate seed-audio-1 "Welcome to VHS." -o welcome.mp3
vhscli generate seed-audio-1 "Read this in the reference voice." -i voice.mp3 -o out.mp3
vhscli generate seed-audio-1 "Blend these voices." -i v1.mp3 -i v2.mp3 -o blend.mp3
```

---

## How a task is tracked — what `generate`, `submit`, and `resume` share

A generation is a row in the VHS backend. The CLI writes no database and no
sidecar files, so **the task id is the only handle to a task** — lose it and
you have stranded something you paid for.

- `generate` submits, waits, and saves to `-o`. It prints the id first, so an
  interrupted run is resumable.
- `submit` prints the id (`task_id: <uuid>`) and exits without waiting. `-o`
  is validated but nothing is written yet.
- `resume <task_id> -o <path>` waits for the task and writes its output.

`-o` is required on all three, including `resume`: the task knows which model
ran and what was asked, but only you know where the file belongs. The CLI
writes exactly to `-o` — it does not re-home or de-conflict, so check the path
first if you must not overwrite.

Everything is keyed by the task id and idempotent, so re-running a command
that died mid-flight joins the existing task rather than paying twice. Use
`--task-id <uuid>` to choose the id yourself when you need it recorded before
the submit returns.

`vhscli chat` has no task to track — it is fast and prints to stdout.

---

## vhscli submit — submit a task and exit (don't wait)

```
vhscli submit <model> <prompt> -o <path> [...same flags as `vhscli generate <model>`]
```

`submit` takes the **same models and the same options** as `generate`
(`seedance-2`, `seedream-5`, `seedream-5-pro`, `nano-banana-2`,
`nano-banana-pro`, `gpt-image-2`, `seed-audio-1`). The only difference is that
once the backend has the task it prints `task_id: <uuid>` and exits without
polling. `-o` is checked but nothing is written until you resume.

Use it when:

- The job is long (e.g. seedance video) and you don't want to keep the
  terminal blocked.
- You want to fan out several tasks in parallel and pull results later.

**Capture the printed id** and pass it to `vhscli resume <task_id> -o <path>`
to fetch the result. Nothing on this machine remembers the task for you.

`--task-id <uuid>` submits under an id you choose instead of a fresh one, so
you can write the id down *before* the submit returns. Use it when losing the
id would strand a paid-for task: mint a uuid, persist it, then submit. If the
command dies before it prints anything, re-run the identical command — the
backend row and the submit are both keyed by that id and idempotent, so the
re-run joins the same task rather than starting a second
one.

Examples:

```
# kick off a video, get the terminal back, finish later
vhscli submit seedance-2 "a robot dancing in tokyo at night" -o robot.mp4
# prints: task_id: 7d3c1b2a-...
# ... do other work ...
vhscli resume 7d3c1b2a-...

# fan out several image jobs, then collect them all
vhscli submit seedream-5 "a red fox in a snowy forest" -o fox.jpg   # task_id: <id1>
vhscli submit seedream-5 "a blue jay on a branch"      -o jay.jpg   # task_id: <id2>
vhscli submit seedream-5 "an orca breaching"           -o orca.jpg  # task_id: <id3>
vhscli resume <id1> <id2> <id3>
```

---

## vhscli resume — finish a submitted generation by task id

```
vhscli resume <task_id> -o <path>
```

Takes the id `submit` printed and the path to write. `resume`:

- Looks the task up on the server (fatal if the id is unknown there).
- Recovers which model ran from the task itself, and checks `-o` matches that
  model's output kind before waiting.
- Waits for the task to finish if it is still running.
- Saves the media to `-o`. The extension sets the saved format; the CLI
  converts if needed. It writes exactly there — no re-homing, no ` (N)`.

Safe to re-run: a finished task is re-read from the server, never resubmitted.
To fan out, run one `resume` per id (in parallel shells if you like) — each is
one task and one file.

When to use `resume`:

- You ran `vhscli submit ...` and now want the result.
- Your `vhscli generate ...` was interrupted (ctrl-c, crash, closed terminal,
  lost network) — it printed the id before it started waiting.

Examples:

```
vhscli resume 7d3c1b2a-... -o fox.png
```

---

## Understanding local images, video, and pdfs

**Do NOT use the Read tool, or any built-in file-reading capability, to "look
at" images, video, or pdfs.** That path either fails or gives you a garbled
snippet. The only correct way to understand local visual or document content
is `vhscli chat` with `-i` / `-v` / `-f`.

```
vhscli chat "what's happening?" -i photo.jpg
vhscli chat "transcribe the speech" -v clip.mp4 --fps 2
vhscli chat "summarize this paper" -f paper.pdf
```

### Prompt patterns for visual / document understanding

`vhscli chat` understands images, pdfs, and video frames, but **not** audio
inside videos. Ask for **structured JSON output** when you'll parse the
answer, and **name every field** you want. Be explicit about formats
(timestamp style, units, language).

Image — describe / classify:

```
vhscli chat "describe the scene. return json {objects:[{label,bbox?}], setting, mood, dominant_colors:[]}." -i photo.jpg
vhscli chat "classify the image into one of: cat, dog, bird, other. return json {label, confidence_0_1, reasoning}." -i pic.jpg
```

Image — OCR / text extraction:

```
vhscli chat "transcribe all visible text verbatim, preserving line breaks and reading order. do not paraphrase." -i receipt.jpg
vhscli chat "extract the receipt as json {merchant, date_iso, items:[{name, qty, unit_price, line_total}], subtotal, tax, total, currency}." -i receipt.jpg
```

Image — comparison (number them in the prompt):

```
vhscli chat "compare image 1 and image 2. return json {same_subject:bool, differences:[], which_is_better, why}." -i a.jpg -i b.jpg
vhscli chat "image 1 is the original, image 2 is an edit. list every visible change as json {changes:[{region, before, after}]}." -i orig.png -i edit.png
```

PDF — summarize / outline (always ask for page anchors):

```
vhscli chat "summarize this paper in 5 bullets. each bullet must include the source page as {page:int, point:string}. return json {bullets:[...]}." -f paper.pdf
vhscli chat "extract the outline as json [{page, heading_level, heading, bullets:[]}]." -f doc.pdf
```

PDF — QA / extraction:

```
vhscli chat "answer using only this document. question: what is the experimental setup? return json {answer, citations:[{page, quote}]}." -f paper.pdf
vhscli chat "extract every table as json [{page, title?, headers:[], rows:[[...]]}]." -f report.pdf
```

Video — events / timeline (state the timestamp format):

```
vhscli chat "list key events. return json [{start_time, end_time, event}]. use HH:mm:ss." -v clip.mp4 --fps 2
vhscli chat "describe the movement sequence and any safety risks. return json [{start_time, end_time, event, danger:'none'|'low'|'med'|'high'}]. HH:mm:ss." -v clip.mp4 --fps 3
```

Video — temporal QA / counting:

```
vhscli chat "at what timestamp does the referee first appear? return json {timestamp_hms, evidence}." -v match.mp4 --fps 2
vhscli chat "count how many distinct people appear. return json {count, per_person:[{first_seen_hms, description}]}." -v scene.mp4 --fps 3
```

Choosing `--fps` for video (default 1, range 0.2–5):

- **3–5** — counting actions, sports, fast cuts, dense motion.
- **1** — general description, dialogue scenes.
- **0.2–0.5** — long static footage, headcount, slow surveillance.

Higher fps = more detail but more tokens and slower. Lower fps = cheaper but
may miss brief events.

## Tips

- Always quote prompts.
- `-o` is required for `vhscli generate` / `vhscli submit` / `vhscli resume`.
  It's relative to your cwd; output format follows the extension and the CLI
  converts if needed. The CLI writes exactly there and will overwrite, so pick
  a free name yourself if that matters.
- Short options accept no-space form: `-ofoo.jpg`. Long options accept `=`:
  `--size=2K`.
- Use `--` to pass a prompt starting with a dash:
  `vhscli generate seedream-5 -o x.jpg -- "-weird prompt"`.
- Reference images (`-i`, `--first-frame`, `--last-frame`) can be any common
  format; non-JPEG/PNG inputs (e.g. HEIC, WebP, TIFF, BMP) are converted to
  JPEG before upload.
- Reference files are uploaded to temporary cloud storage for the provider to
  fetch, and expire there on their own. Each run re-uploads them, so passing a
  large reference repeatedly costs an upload each time.
- Unknown command? `vhscli` will suggest the closest match.


## --json (scripting)

Add `--json` to `generate`, `submit`, or `resume` to get one JSON object per
line on stdout while logs go to stderr:

```
{"event":"submitted","task_id":"..."}
{"event":"progress","task_id":"...","elapsed_s":42}
{"event":"done","task_id":"...","path":"/abs/out.mp4","hash":"sha256:...","size":123}
{"event":"error","message":"..."}
```

- `submitted` fires as soon as the backend accepts the job — persist the
  task_id and you can `vhscli resume <task_id> -o <path> --json` later, even
  after a crash, from any machine you are logged in on. The id is the whole
  handle; nothing local is needed.
- `done.path` is the file written (always the `-o` you asked for); `hash` is
  sha256 of its content.
- a failing command emits `error` and exits 1.

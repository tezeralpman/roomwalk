---
name: roomwalk
description: >-
  Build a scroll-driven walk through a generated interior for a maker's website —
  the camera walks past the things the business actually makes, stops at each one,
  and the shop's real photographs surface beside it. Generates the footage through
  the Higgsfield MCP, slices it to frames with AVFoundation (no ffmpeg needed),
  measures its own seams, and wires the scroll. Use for "scroll video walkthrough",
  "camera walks through a room on scroll", "Apple-style scroll hero", "hero video
  scrubbed by scroll", "interior walk for my furniture/joinery/kitchen/stone site",
  or when a scroll-driven hero has abrupt cuts, teleports, objects growing out of
  nothing, or steppy scrubbing that needs diagnosing.
---

# roomwalk

A scroll hero where the camera walks through one room and stops at each thing the
business makes. Built for makers — joinery, furniture, kitchens, stone, glass —
where the product is physical and the shop already owns photographs of real work.

The generated interior is the **stage**. The shop's real photographs are the
**proof**, and they surface at the stop that shows their category. That split is
what keeps the page honest: nobody is shown a rendered object and told it was built
in the workshop.

---

## What you need before starting

1. **The Higgsfield MCP connector.** If the `generate_video` / `generate_image` tools
   are not in this session, connect it before anything else:

   ```bash
   python3 "${CLAUDE_PLUGIN_ROOT}/skills/roomwalk/tools/connect_higgsfield.py"
   ```

   Run it in the background, read its output, and hand the user the sign-in link it
   prints on its own line. It registers a client, waits for the browser callback,
   writes the token into the user-scope MCP config, and asks for a restart. One click
   from the user, nothing else.

   Do **not** reach for `claude mcp login higgsfield` — it always fails. Higgsfield's
   metadata declares `issuer: https://mcp.higgsfield.ai` while the redirect carries
   `iss=https://clerk.higgsfield.ai`, and Claude Code rejects that per RFC 9207, with no
   flag to disable the check. Their advertised device-code server returns 404 on every
   endpoint, so that route is dead too. The script exists because of both.

   When the stored token expires, `--refresh` renews it without a new sign-in;
   `--status` reports what is currently configured.
2. **Swift.** `swiftc` ships with Xcode Command Line Tools. The frame tools are
   AVFoundation — ffmpeg is not required and is usually absent on a designer's Mac.
3. **The client's real photographs.** Pulled from their own site in step 2. Without
   them this skill has no proof layer and should not be run.
4. **Credits.** A five-stop draft costs roughly 60–80 credits at 480p. Always
   preflight with `get_cost: true` before a final pass.

If the connector cannot be established, say so and stop. Do not fall back to the REST
API unless the user has keys and asks for it.

---

## The pipeline

### 1 · Ask for the link, and ask for nothing else

Open with one line:

> *Пришлите ссылку на сайт мастерской — я посмотрю, что вы делаете, и предложу план.*
> *(Send me the link to the shop's site — I'll read it and propose a plan.)*

Do not ask about stops, tone, colours, or anything else yet. Everything you need is
usually on their existing site, and asking a person to describe what their own site
already says is the fastest way to lose them.

### 2 · Read the site

Fetch the page, then the pages it links to. What you are looking for:

- **Categories of work**, and how many photographs sit behind each. A gallery with 155
  photos and one with 3 are not equal candidates for a stop.
- **The photographs themselves.** Note the pattern. Many shop sites serve a thumbnail
  (`k21s.jpg`) beside a full-size file at the same path without the suffix (`k21.jpg`) —
  try dropping the suffix before settling for thumbnails.
- **Prices**, if any. An inline price table or a calculator's JavaScript is gold: it lets
  the page show a real "from N" instead of "contact us".
- **The differentiator.** Usually one sentence they repeat: *"we don't sell panels, we
  make things out of boards."* That line is the hero, not something you invent.
- **Contacts, founding year, ratings.** Real numbers for the proof strip — never invent
  one, and see gate: an invented metric is worse than no metric.

Then download 8–10 photographs per category into `assets/catalog/<slug>/` and write a
`catalog.json` recording each category's title, its files, and its **true total** on the
source site. The counter matters: "все 155 →" is a different promise from "все 8 →".

**If the site cannot be read** — JavaScript-only shell, auth wall, dead domain, a
one-pager with no gallery — say so plainly and ask for exactly two things:

> *Не смог прочитать сайт — там нечего разобрать. Расскажите в двух словах, что делает
> мастерская, и пришлите несколько фотографий готовых работ.*

Do not guess the business from its domain name. Do not proceed without photographs: a
walk with no proof layer is a rendered fantasy with a phone number on it.

### 3 · Propose the plan, then wait

Come back with a short, concrete proposal — what you found, what you would build, what it
costs. Not a questionnaire. Something like:

> Посмотрел сайт. Мастерская делает столешницы (155 фото), подоконники (118), столы (55),
> шкафы (32) и ещё пять направлений — всего 618 работ. Есть калькулятор с настоящими
> ценами: от 14 300 ₽/м². Работает с 2003 года, рейтинг 5,0 в Яндексе.
>
> Предлагаю проход по одной комнате с пятью остановками: столешница → подоконник →
> оформление окна → стол → шкаф, и в конце шкаф открывается. На каждой остановке
> всплывают ваши настоящие фотографии этой категории и ведут в каталог.
>
> Черновик в 480p — около 70 кредитов, минут двадцать. Финал в 1080p — ещё 290.
>
> Делаем так?

Three things this must contain: **the stops mapped to their real categories with counts**,
**the cost**, and **one question at the end**. Wait for a yes before spending anything.

If they want different stops, take theirs — but hold the one-room rule and the four-to-six
range. More than six stops means a second room, and § One room explains why that tears.

### 4 · Establish the room

Generate one wide establishing shot with `generate_image` (`nano_banana_pro`, 2k,
16:9) that **names the layout explicitly by wall**:

> *"Wide establishing interior photograph of one open-plan room, morning light.
> Layout, left to right: an oak worktop runs along the LEFT wall; a large window
> with a thick sill is on the FAR wall straight ahead; a dining table stands in the
> MIDDLE; a tall wardrobe is against the RIGHT wall next to an open doorway."*

Look at the result. If the layout does not match what you asked for, regenerate —
everything downstream inherits this geometry. Two credits is cheap; a broken walk
is not.

### 5 · Cut the anchors

For each stop, generate a still **from the establishing shot as `image_references`**.
These are the frames the camera must arrive at. Use `generate_image_batch` — they are
independent and run in parallel.

Anchors are the whole trick. Without them the model invents the arrival, and you get
the two classic failures: **an object grows into frame that was not there before**,
and **the camera reverses** to find its subject.

### 6 · Generate the transitions

**The rule that matters: lock both ends.**

- `start_image` — where the camera is now
- `end_image` — the next anchor

Prompt for a person holding a camera, not a drone:

> *"Handheld camera walks slowly forward along the worktop toward the window. The
> worktop edge slides out of the bottom of frame; the sill fills the view. One
> continuous handheld take, natural walking gait, slight organic sway, steady
> morning daylight. Camera only moves forward, never backwards. No cuts, no people."*

Name what **leaves** the frame and in which direction. That single habit prevents
most spatial nonsense — it tells the model where things are relative to the camera.

Model: `seedance_2_0_mini` at 480p for drafts, 8 s per segment. Sixteen models accept
both `start_image` and `end_image`; query the catalogue with `models_explore` if you
need 4K or a longer take.

**Chain from the real tail.** After the first segment renders, extract its actual last
frame and upload that as the next segment's `start_image` — do not reuse the pristine
anchor. The model rarely lands exactly on the anchor, and starting the next segment
from the anchor puts a visible jump at the join. Starting from the real tail makes the
join continuous by construction while `end_image` still controls the arrival.

Batch what you can: transitions whose start frames already exist run in parallel.
Chained ones are sequential by nature.

> Higgsfield sometimes answers a submission with a preset recommendation instead of a
> job. Resubmit with `declined_preset_id` set to the id it returned. It also
> rate-limits bursts — wait and retry rather than dropping the segment.

### 7 · One room

**Do not cross into a second room.** This is the single biggest cause of torn walks.

Measured on a real build: transitions inside one room joined at 1.07–1.86× the normal
frame-to-frame change — invisible. The three transitions that walked through a doorway
into a second room tore at 6.3×, 11.4× and 8.4×, because the model could not reach an
`end_image` that showed an entirely different space in eight seconds.

If the brief needs more categories than one room holds, put the extras in the catalogue
below the hero. A shop's real photographs of a chest of drawers beat a generated one.

### 8 · Slice to frames

```bash
swiftc -O tools/extract_frames.swift -o extract_frames
./extract_frames segment.mp4 ./out --count 120 --width 864 --quality 0.66
```

**120 frames per 8-second segment.** Sixty is not enough — it samples at 7.5 fps and
scrubbing shows steps. The tool writes `manifest.json` beside the frames.

Two traps it already handles, both of which cost a rebuild to find:

- `requestedTimeToleranceBefore/After = .zero`, or `AVAssetImageGenerator` returns the
  nearest keyframe and you get duplicate frames.
- Frame index comes from `requestedTime`, not a call counter — the async callbacks do
  **not** arrive in request order, and a naive counter shuffles the sequence.

Concatenate the segments into one numbered run and write a combined manifest.

### 9 · Measure the seams

```bash
swiftc -O tools/measure_seams.swift -o measure_seams
./measure_seams ./frames 120
```

Compares the gap at each join against the median frame-to-frame change inside the
segments. Read it as: **under 2× invisible, 2–5× slightly visible, over 5× torn**.

A torn seam means that segment never reached its anchor. Regenerate it chained from
the real tail. A slightly-visible seam can be softened without spending credits:

```bash
swiftc -O tools/blend_seam.swift -o blend_seam
./blend_seam ./frames 480 20      # frame index of the join, blend width
```

Blending caps out around 2.5× — it cannot rescue a torn seam, only polish a mild one.

Also available: `measure_flicker` reports per-frame brightness jitter and how far the
last frame sits from the first (the loop seam). Jitter invisible at 30 fps reads as
flicker when scrubbed slowly. If the material has strong texture and raking light, run
Higgsfield's `video_deflicker` **before** slicing.

### 10 · Wire the scroll

Copy `web/scroll_frames.js`. It draws to `<canvas>` — never swap `<img src>`, which
re-decodes and flashes.

**Section height governs smoothness.** Target **9–10 px of scroll per frame**:

```
walk height in screens ≈ (frames × 10 / viewport height) + 1
```

Six hundred frames on an 833 px viewport → about 8 screens. The same 600 frames
stretched over 15 screens gives 28 px per frame and visibly steps.

Pacing lives in the `timeline` option, not in the footage. Generate an even camera
move, then make the rhythm here:

```js
timeline: [
  { to: 0.030, scroll: 3 },   // dwell at the first stop
  { to: 0.170, scroll: 2 },   // travel
  { to: 0.230, scroll: 3 },   // dwell
  // …
]
```

`to` is how far through the frames to reach; `scroll` is how much scroll to spend
getting there. Weights are normalised, so only their ratios matter. Dwell 3 against
travel 2 gives roughly a fourfold speed difference — measured at 2.5–3.7 frames per
1 % of scroll on travel against 0.08–0.22 on a dwell. Retuning the rhythm is five
numbers, never a regeneration.

Put the stops at exact segment boundaries — `k / segmentCount` — so a caption never
lands mid-move.

### 11 · Surface the real work

At each stop, fade in three of the shop's real photographs from that category, and
make them **clickable** — they should open that section of the catalogue. Decorative
thumbnails that ignore a click are worse than no thumbnails.

At the final stop, show a grid drawn from several categories: the walk ends by opening
onto everything the shop does.

---

## Verify before handing back

The pane a preview renders in may not fire scroll events or tick
`requestAnimationFrame`, and its screenshots can come back dark regardless of the
page. Verify with numbers, not with a glance:

- **Frames differ across scroll.** Hash the canvas at ~20 scroll positions; every
  position should differ. If they are identical, the frames never loaded or the
  listener never attached.
- **Stops fire in order.** Trace the caption across the same sweep.
- **Seams pass.** `measure_seams` under 2× everywhere, or a stated reason why not.
- **Px per frame.** Section travel ÷ frame count, target 9–10.
- **No horizontal scroll** at 320 / 375 / 414 / 768 px.
- **Look at the composite.** `preview_hero` renders a frame with the page's gradients
  and caption into a PNG, bypassing screen capture entirely. Use it when the preview
  looks wrong but the measurements look right.

Never report the walk as working on the strength of the code alone. On a real build,
every functional check passed while the hero was showing frames from an entirely
different video — the copy had nested one frame directory inside another. Check what
is **on** the canvas, not only that a canvas exists.

---

## Weight

Six hundred frames at 864 px JPEG is about 32 MB. Fine for a prototype, heavy for
production. In order of effect: WebP or AVIF instead of JPEG (−30–50 %), narrower
frames stretched by CSS, fewer frames on mobile, a sprite atlas to collapse the
request count. Or scrub a real `<video>` element — a 30-second 1080p H.264 is roughly
4 MB, six times lighter, at the cost of seek precision in Safari. Say the number out
loud before the client discovers it.

---

## Honesty

The interior is generated; the furniture in it was never built by anyone. Keep the
shop's real photographs as the proof layer, keep them at the stop that matches their
category, and put a plain line in the footer saying the interior is styling rather
than portfolio. A visitor who orders "the table from your homepage" and learns it does
not exist is a worse outcome than a slightly less cinematic hero.

# roomwalk

**A scroll hero where the camera walks through one room and stops at each thing the business makes.**

Built for makers — joinery, furniture, kitchens, stone, glass — where the product is
physical and the shop already owns photographs of real work.

The generated interior is the **stage**. The shop's real photographs are the **proof**,
and they surface at the stop that shows their category. That split is what keeps the
page honest: nobody is shown a rendered object and told it was built in the workshop.

---

## Install

Paste this into Claude Code and press enter. Nothing else to set up.

> Install the roomwalk plugin: run `claude plugin marketplace add tezeralpman/roomwalk`
> then `claude plugin install roomwalk@roomwalk`. After that connect Higgsfield by
> running the plugin's `connect_higgsfield.py` in the background and giving me the
> sign-in link it prints.

It installs the plugin, starts the Higgsfield sign-in, and hands you a link. Click it,
log into your Higgsfield account, and the token writes itself into your MCP config.
Restart Claude Code once and everything is live.

Prefer doing it by hand? Two commands and one script:

```
/plugin marketplace add tezeralpman/roomwalk
/plugin install roomwalk@roomwalk
/roomwalk:connect
```

Then just say what you want:

> *Сделай скролл-проход по комнате для сайта столярной мастерской. Фотографии работ — на woodenwolf.ru*

> *Build a scroll walkthrough hero for my kitchen studio. Stops: worktop, sink run, island, cabinet fronts.*

The skill handles the rest — plans the stops, generates the room, cuts the anchors,
renders the transitions, slices the frames, measures its own seams, and wires the scroll.

---

## What you need

| | |
| --- | --- |
| **Higgsfield MCP** | Add `https://mcp.higgsfield.ai/mcp` through the connector UI at claude.ai. **Not** via `claude mcp login` — see below. |
| **Swift** | `swiftc`, from Xcode Command Line Tools. The frame tools are AVFoundation; ffmpeg is not needed. |
| **Real photographs** | The client's own finished work. Without them there is no proof layer. |
| **Credits** | A five-stop draft runs about 60–80 credits at 480p. |

### The connector, specifically

`claude mcp login higgsfield` always fails with *"Issuer mismatch in authorization
response (RFC 9207)"*. Higgsfield's metadata declares `issuer: https://mcp.higgsfield.ai`
while the redirect carries `iss=https://clerk.higgsfield.ai` from their upstream Clerk.
Claude Code rejects that correctly and there is no flag to disable the check. Their
advertised device-code server, `fnf-device-auth.higgsfield.ai`, returns 404 on every
endpoint, so that route is dead as well.

`/roomwalk:connect` works around it. It runs the same authorization-code + PKCE flow with
the same `state` check and skips only the `iss` comparison — a mix-up protection that
matters when a client talks to several authorization servers, where here there is exactly
one, taken from Higgsfield's own metadata. Then it writes the token into the user-scope
MCP config as a static bearer header.

Because the header is static, the token eventually expires. `--refresh` renews it without
a new sign-in; `--status` reports what is configured. Adding the connector through the
claude.ai UI also works if you would rather click than run a script.

---

## What's inside

```
skills/roomwalk/
  SKILL.md                    the pipeline, the rules, the failure modes
  tools/extract_frames.swift  video → numbered frames + manifest (AVFoundation)
  tools/measure_seams.swift   how visible is each join, in numbers
  tools/measure_flicker.swift per-frame brightness jitter and loop-seam distance
  tools/blend_seam.swift      soften a mild join without spending credits
  tools/preview_hero.swift    composite a frame with gradients + caption to PNG
  tools/connect_higgsfield.py Higgsfield sign-in that works around the RFC 9207 bug
  web/scroll_frames.js        canvas scroll engine with a pacing timeline
commands/
  connect.md                  /roomwalk:connect
  walk.md                     /roomwalk:walk
```

---

## The three rules that decide whether it works

**Lock both ends.** Every transition gets a `start_image` *and* an `end_image`. Without
the end anchor the model invents the arrival, and you get the two classic failures: an
object grows into frame that was not there before, and the camera reverses to find its
subject.

**Chain from the real tail.** The model rarely lands exactly on the anchor. Start the
next segment from the previous segment's *actual* last frame, not the pristine anchor —
the join becomes continuous by construction while `end_image` still controls arrival.

**One room.** Measured on a real build: transitions inside one room joined at 1.07–1.86×
the normal frame-to-frame change, invisible. The three that walked through a doorway into
a second room tore at 6.3×, 11.4× and 8.4×. If the brief needs more categories than one
room holds, put the extras in the catalogue below the hero.

---

## Two numbers to hit

**120 frames per 8-second segment.** Sixty samples at 7.5 fps and scrubbing shows steps.

**9–10 px of scroll per frame.** Section height in screens ≈ `(frames × 10 / viewport) + 1`.
Six hundred frames on an 833 px viewport is about 8 screens. The same frames stretched
over 15 screens gives 28 px per frame and visibly steps.

Pacing is not shot — it lives in the `timeline` option. Generate an even camera move,
then set the rhythm in five numbers. Dwell 3 against travel 2 gives roughly a fourfold
speed difference, and retuning never costs a regeneration.

---

## Weight, said out loud

Six hundred frames at 864 px JPEG is about 32 MB. Fine for a prototype, heavy for
production. WebP or AVIF takes 30–50 % off; narrower frames stretched by CSS take more;
a sprite atlas collapses the request count. Or scrub a real `<video>` — a 30-second 1080p
H.264 is roughly 4 MB, six times lighter, at the cost of seek precision in Safari.

---

## Honesty

The interior is generated; the furniture in it was never built by anyone. Keep the shop's
real photographs as the proof layer, keep them at the stop that matches their category,
and put a plain line in the footer saying the interior is styling rather than portfolio.
A visitor who orders "the table from your homepage" and learns it does not exist is a
worse outcome than a slightly less cinematic hero.

---

MIT.

---
name: exploded
description: >-
  Build a scroll-driven product hero from a single unbroken shot on pure black — the object
  sits assembled, and as you scroll it comes apart along its own axes into a floating
  exploded view. One take, no cuts, no seams to measure, because there is only ever one
  segment. Generates through the Higgsfield MCP with the first and last frame both locked,
  slices frames with AVFoundation (no ffmpeg), and scrubs them on a canvas. Use for "product
  landing page", "scroll explodes the product", "Apple-style product hero", "exploded view on
  scroll", "watch/bottle/shoe/tool landing page", or whenever a single object is the whole
  story and a walkthrough would be overkill.
---

# exploded

One object on pure black. Scroll, and it takes itself apart.

This is the sibling of `roomwalk` and the simpler of the two. Where a walk crosses a space
in several takes and every join has to be anchored, measured and sometimes rescued, this is
**one continuous shot**. There is nothing to join. The failure mode that costs the most time
in the walk — a torn seam — cannot occur here.

Reach for it when a **single object carries the offer**: a watch, a bottle of scent, a
sneaker, a chair, a hand plane, a coffee grinder, a jar of cream, a lock, a lamp. Reach for
`roomwalk` instead when the offer is a **range** and the visitor needs to see several things
in the place they belong.

---

## Why the black background is not a style choice

Every asset sits on pure black, and so does the page. That means the hero has **no edge**. A
walkthrough hero ends at a hard line where the footage stops and the page begins, and that
line reads as a wall — bridging it takes a gradient and still shows. Here the frames simply
dissolve into the page ground, because they are the same colour.

It also makes the frames compress far better: large flat black areas cost almost nothing in
JPEG, so a 700-frame sequence lands much lighter than the same count of a lit interior.

---

## What you need before starting

1. **The Higgsfield MCP connector.** If `generate_video` / `generate_image` are not in this
   session, connect it first:

   ```bash
   python3 "${CLAUDE_PLUGIN_ROOT}/skills/roomwalk/tools/connect_higgsfield.py"
   ```

   Hand the user the sign-in link it prints. `claude mcp login higgsfield` does not work —
   see the `roomwalk` skill for why.
2. **Swift**, for the frame tools. `swiftc` ships with Xcode Command Line Tools.
3. **A photograph of the real product**, if the product is real. See § Honesty.
4. **Credits.** A single 10-second draft at 480p is around 10 credits; the 1080p final is
   about 90. Preflight with `get_cost: true`.

---

## The pipeline

### 1 · Ask for the product, and for a photograph of it

> *Что показываем? Пришлите фотографию изделия — лучше на простом фоне — и я соберу героя,
> в котором оно разбирается на части по скроллу.*

If the product is real and purchasable, **its photograph must come from the client**. If it
is a concept, a category illustration, or the client explicitly has nothing, generate the
base image and say plainly in the handover that the object shown is a representation.

### 2 · Asset 1 — the object, assembled

If the client supplied a photograph, put it on black first — `remove_background`, then
composite onto `oklch(0% 0 0)`. If you are generating it, ask `nano_banana_pro` for exactly
this shape of image:

> *"A studio-grade product photograph of <object>, shown at a three-quarter angle revealing
> both <face> and <side>. Pure black background with zero ambient light bleed, no reflections
> on the surface, no cast shadow. The object is fully assembled. Shot as if for a high-end
> print campaign — clinical precision, no stylisation."*

The three phrases that matter: **pure black background**, **no ambient light bleed**, **no
cast shadow**. Without them you get a floor plane, and a floor plane makes the object look
like it is standing in a room rather than floating in the page.

### 3 · Asset 2 — the same object, apart

Generate with **asset 1 as `image_references`**, never from scratch. This is what keeps the
parts recognisably the same object:

> *"Using the provided reference: deconstruct the object into a precise exploded-view
> diagram. Each component — <name them, in order along the axis> — floats apart from its
> assembled position along its natural mechanical axis, with uniform spacing. Deliberate and
> symmetrical, like a technical illustration. Pure black background. All parts keep their
> original finish and material. No labels, no lines, no graphic overlays."*

**Name the components yourself.** "Deconstruct it" gives you shrapnel; naming the bezel, the
crystal, the dial, the movement plate, the caseback gives you an exploded view. For a
worktop: the boards, the glue lines, the reinforcing splines, the edge profile, the finish
layer. For a bottle: the cap, the collar, the atomiser, the tube, the glass, the base.

Look at the result. If the parts do not read as the same object, regenerate — everything
downstream inherits this.

### 4 · Asset 3 — one shot, both ends locked

**This is the whole trick, and it is one call:**

- `start_image` — asset 1, the assembled object
- `end_image` — asset 2, the exploded arrangement

```
model: seedance_2_5     duration: 10–20 s     resolution: 480p draft / 1080p final
aspect_ratio: 16:9      generate_audio: false
```

The prompt describes the *journey between two frames the model already has*, so it can only
choose how to get there:

> *"The object floats in a pure black void, fully assembled, with no environment, no ground
> plane, no ambient reflections. The camera begins at a front-right angle and orbits slowly
> clockwise in one smooth uninterrupted arc. About halfway through, the object begins a
> seamless mechanical deconstruction: each component separates along its natural axis with
> deliberate, weighted momentum. Parts drift outward in radial symmetry, as if gravity had
> been selectively reversed. Slow, cinematic and precise — never chaotic. By the end all
> parts hang in a balanced exploded arrangement, still against the black void."*

Because both ends are pinned, the model cannot invent an arrival, cannot reverse to find its
subject, and cannot grow a new element into frame — the three failures that cost the most in
a multi-segment walk. **There are no seams to measure. Skip the seam tooling entirely.**

`seedance_2_5` reaches 30 seconds, which is far more than this needs. Ten seconds at 24 fps
is 240 source frames — plenty for a 700-frame sequence after slicing.

### 5 · Slice, then stabilise

```bash
swiftc -O ../roomwalk/tools/extract_frames.swift -o extract_frames
./extract_frames hero.mp4 ./frames --count 240 --width 1280 --quality 0.7
```

On black, the frames are small — a flat background costs almost nothing — so you can afford
more of them and a wider frame than a walkthrough allows.

Then run the exposure pass anyway:

```bash
swiftc -O ../roomwalk/tools/stabilise_exposure.swift -o stabilise_exposure
./brightness_curve ./frames        # look first
./stabilise_exposure ./frames      # then correct
```

A single take drifts less than five stitched ones, but the model still brightens the object
as it opens up. On black that reads as the object glowing on its own, which is worse than it
sounds — it looks like a rendering artefact rather than light.

### 6 · Scrub it

Copy `../roomwalk/web/scroll_frames.js`. Same engine, but here the setup is simpler because
there are no stops:

```js
new ScrollFrames({
  canvas: document.querySelector('#hero'),
  scroller: document.querySelector('.hero-section'),
  dir: './frames',
  manifest: './frames/manifest.json',
  // no timeline — one continuous move deserves a continuous, even mapping
});
```

**Leave `timeline` out.** A walk needs holds because it visits places; a deconstruction is a
single gesture, and pausing it mid-way reads as a stall. Even mapping, start to finish.

Section height at **9–10 px of scroll per frame**, as always: 240 frames on an 833 px
viewport is about 4 screens. Shorter than a walk, and that is correct — one gesture should
not take fifteen screens to complete.

Page ground must be the same black as the frames, or the edge comes back.

### 7 · The rest of the page

Black throughout. The sections that suit this shape, in order:

- **Overlay on the hero** — a small label, one display line, one sentence, one button. Sitting
  over a bottom gradient so it reads at any frame. Nothing else.
- **Features** — what the parts you just watched separate actually do. The exploded view has
  earned the reader's attention on components; spend it. Real copy, no lorem.
- **Specifications** — a two-column table. This is where an exploded hero pays off: the reader
  has just seen the anatomy, so the numbers land.
- **Closing CTA** — one line, one button.

Use `useInView`-style fade-ins if the project has a motion library; plain
`IntersectionObserver` with an opacity and 24 px rise otherwise. One entrance pattern for the
whole page, not one per section.

---

## Verify before handing back

- **Frames differ across scroll.** Hash the canvas at ~20 positions; all should differ.
- **No stall.** No two adjacent samples identical — that means a frame never loaded.
- **Px per frame** between 9 and 10.
- **The background is the same black** in the frames and in the page. Sample a corner pixel of
  the canvas and compare with the computed page background; they should match within a point
  or two. If the frames are `#0a0a0a` and the page is `#000`, the edge shows on a good screen.
- **No horizontal scroll** at 320 / 375 / 414 / 768 px.
- **`brightness_curve` drift** under about ±6 across the take.

---

## Honesty

An exploded view is a legitimate and old convention in product marketing — nobody believes a
watch really floats apart. That is not the risk here.

The risk is the **object itself**. If the product is real and someone can buy it, the base
image must be the client's own photograph. Generating a plausible-looking version of a real
product and exploding that is misrepresentation: the proportions, the finish, the component
count will all be subtly wrong, and the page is making a claim about a thing that exists.

- **Client's product, client's photograph.** Generate the explosion from it, never the product.
- **No third-party brands.** Do not generate a recognisable Rolex, Dyson, or anything else the
  client does not own, even as a demo.
- **No fabricated specifications.** The specs table is where an exploded hero tempts you to
  invent a movement type or a power reserve. Every number comes from the client.
- **Regulated products** — medicines, supplements, cosmetics with active claims, anything with
  a dosage — get the hero and nothing else. Composition and effect claims come from their own
  approved material.

If the client has no photograph and wants the object generated anyway, build it and say once,
plainly, in the handover: the object shown is a representation, not their product.

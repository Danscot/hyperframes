<!-- Omitted missing pages: /concepts/clips-and-tracks, /concepts/timing-and-duration, /guides/captions, /guides/transitions -->

You are an expert HyperFrames composition author. Output a single self-contained HTML document that conforms to the rules below.

## HARD RULES (violations break rendering)

1. Do not animate `width`, `height`, `top`, or `left` directly on `<video>` elements; wrap video in a `<div>` and animate the wrapper.
2. Do not control media playback in scripts (`video.play()`, `video.pause()`, `audio.currentTime`); the framework controls playback from data attributes.
3. Composition duration is determined by GSAP timeline duration; extend the timeline (for example with `tl.set({}, {}, TIME)`) so media is not cut off.
4. Timed visible elements must include `class="clip"` so runtime visibility lifecycle works.
5. Use appropriately sized source images (roughly up to 2x canvas dimensions) to avoid decode/memory/performance failures.

## /reference/html-schema

---
title: HTML Schema Reference
description: "Complete reference for authoring Hyperframes HTML compositions."
---

This is the full schema reference for Hyperframes compositions. For a gentler introduction, see [Compositions](/concepts/compositions) and [Data Attributes](/concepts/data-attributes).

## Overview

Hyperframes uses HTML as the source of truth for describing a video:

- **HTML clips** = video, image, audio, composition
- **[Data attributes](/concepts/data-attributes)** = timing, metadata, styling
- **CSS** = positioning and appearance
- **GSAP timeline** = animations and playback sync (see [GSAP Animation](/guides/gsap-animation))

## Framework-Managed Behavior

The framework reads data attributes and automatically manages:

- **Primitive clip timeline entries** — reads `data-start`, `data-duration`, and `data-track-index` from clips and adds them to the GSAP timeline
- **Media playback** (play, pause, seek) for `<video>` and `<audio>`
- **Clip lifecycle** — clips are mounted/unmounted based on `data-start` and `data-duration`
- **Timeline synchronization** — keeps media in sync with the GSAP master timeline
- **Media loading** — waits for all media to load before resolving timing

Mounting/unmounting controls **presence**, not appearance. Transitions (fade in, slide in) are animated in scripts.

<Warning>
  Do not manually call `video.play()`, `video.pause()`, set `audio.currentTime`, or mount/unmount clips in scripts. The framework owns media playback and clip lifecycle. See [Common Mistakes](/guides/common-mistakes) for more details.
</Warning>

## Viewport

Every composition must include `data-width` and `data-height` on the root element:

```html
<div id="main" data-composition-id="my-video"
     data-start="0" data-width="1920" data-height="1080">
  <!-- clips -->
</div>
```

Common sizes:
- **Landscape**: `data-width="1920" data-height="1080"`
- **Portrait**: `data-width="1080" data-height="1920"`

## All Clip Attributes

| Attribute | Applies To | Required | Description |
|-----------|-----------|----------|-------------|
| `id` | All | Yes | Unique identifier (e.g., `"el-1"`). Used for relative timing references and CSS targeting. |
| `class="clip"` | Visible elements | Yes | Enables runtime visibility management. Omit for audio-only clips. |
| `data-start` | All | Yes | Start time in seconds (e.g., `"0"`, `"5.5"`), or a clip ID reference for [relative timing](#relative-timing) (e.g., `"intro"`). |
| `data-duration` | video, img, audio | See below | Duration in seconds. **Required** for images. Optional for video/audio (defaults to source duration). Not used on compositions. |
| `data-track-index` | All | Yes | Timeline track number. Controls z-ordering (higher = in front). Clips on the same track cannot overlap. |
| `data-media-start` | video, audio | No | Playback offset / trim point in source file (seconds). Default: `0`. See [Data Attributes](/concepts/data-attributes). |
| `data-volume` | audio, video | No | Volume level from `0` to `1`. Default: `1`. |
| `data-composition-id` | div | On compositions | Unique composition ID. Must match the key used in `window.__timelines`. |
| `data-composition-src` | div | No | Path to external composition HTML file (for [nested compositions](#composition-clips)). |
| `data-variable-values` | div | No | JSON object of values passed to a nested composition. The framework carries the values through, but your composition script must read and apply them manually. |
| `data-width` | div | On compositions | Composition width in pixels. |
| `data-height` | div | On compositions | Composition height in pixels. |

## Clip Types

<AccordionGroup>
  <Accordion title="Video Clips">
    Video clips embed `<video>` elements with timing and playback attributes.

    ```html
    <video
      id="el-1"
      data-start="0"
      data-duration="15"
      data-track-index="0"
      data-media-start="0"
      src="./assets/video.mp4"
    ></video>
    ```

    **Key behavior:**
    - `data-duration` is **optional** — defaults to the remaining duration of the source file from `data-media-start`
    - If source media runs out before `data-duration`, the clip shows the last frame (freeze frame)
    - `data-media-start` trims the beginning of the source video — `data-media-start="5"` starts playback 5 seconds into the source file
    - `data-volume` controls the audio volume of the video — set to `"0"` for silent video
    - Do **not** add `class="clip"` to video elements — the framework manages their visibility directly

    <Warning>
      Do not animate `width`, `height`, `top`, or `left` directly on `<video>` elements with GSAP. This can cause Chrome to stop rendering video frames. Wrap the video in a `<div>` and animate the wrapper instead. See [Common Mistakes](/guides/common-mistakes).
    </Warning>
  </Accordion>

  <Accordion title="Image Clips">
    Image clips display static images with controlled timing.

    ```html
    <img
      id="el-2"
      class="clip"
      data-start="5"
      data-duration="4"
      data-track-index="1"
      src="./assets/overlay.png"
    />
    ```

    **Key behavior:**
    - `data-duration` is **required** for images (unlike video/audio, there is no source duration to default to)
    - `class="clip"` is **required** — this enables the runtime to show/hide the image based on timing
    - Supported formats: PNG, JPG, WebP, SVG, GIF (first frame only)
    - Position and size with CSS — the image renders at its natural size unless styled otherwise
  </Accordion>

  <Accordion title="Audio Clips">
    Audio clips add sound to the composition without any visual element.

    ```html
    <audio
      id="el-4"
      data-start="0"
      data-duration="30"
      data-track-index="2"
      src="./assets/music.mp3"
    ></audio>
    ```

    **Key behavior:**
    - `data-duration` is **optional** — defaults to the remaining duration of the source file from `data-media-start`
    - Audio clips are invisible — do not add `class="clip"` (there is nothing to show/hide)
    - `data-volume` controls volume — use `"0.5"` for background music at 50% volume
    - `data-media-start` trims the beginning of the audio source, just like video
    - Multiple audio clips can overlap on different tracks for layered sound design
  </Accordion>

  <Accordion title="Composition Clips (Nested)">
    Composition clips embed one composition inside another, enabling modular, reusable video building blocks.

    ```html
    <div
      id="el-5"
      data-composition-id="intro-anim"
      data-composition-src="compositions/intro-anim.html"
      data-start="0"
      data-track-index="3"
    ></div>
    ```

    **Key behavior:**
    - Compositions do **not** use `data-duration` — duration is determined by the composition's GSAP timeline (`tl.duration()`)
    - External compositions are loaded from `data-composition-src` and wrapped in `<template>` tags
    - Each nested composition has its own `window.__timelines` entry, registered by its own `<script>` block
    - The framework automatically nests sub-timelines — do not manually add them to the parent timeline
    - Any composition can be nested inside any other — there is no special "root" type
    - Per-instance values can be passed with `data-variable-values`, but the nested composition must read and apply those values itself

    For more on how compositions work, see [Compositions](/concepts/compositions).
  </Accordion>
</AccordionGroup>

## Relative Timing

Reference another clip's ID in `data-start` to mean "start when that clip ends":

```html
<video id="intro" data-start="0" data-duration="10" data-track-index="0" src="..."></video>
<video id="main" data-start="intro" data-duration="20" data-track-index="0" src="..."></video>
```

`main` starts at second 10 (when `intro` ends).

**Offsets** let you add gaps or overlaps:

```html
<!-- 2-second gap after intro -->
<video id="main" data-start="intro + 2" data-duration="20" data-track-index="0" src="..."></video>

<!-- 0.5-second overlap with intro -->
<video id="main" data-start="intro - 0.5" data-duration="20" data-track-index="0" src="..."></video>
```

For a deeper explanation, see the [relative timing section](/concepts/data-attributes#relative-timing) in the Data Attributes concept page.

## Timeline Contract

The framework initializes `window.__timelines = {}` before any scripts run. Every composition must register a GSAP timeline at the key matching its `data-composition-id`:

```javascript
const tl = gsap.timeline({ paused: true });

// Add animations
tl.to("#title", { opacity: 1, duration: 0.5 }, 0);
tl.to("#title", { opacity: 0, duration: 0.5 }, 4.5);

// Register the timeline
window.__timelines["<data-composition-id>"] = tl;
```

### Rules

- Every composition needs a `<script>` block that creates and registers its timeline
- All timelines must start paused (`{ paused: true }`)
- The framework auto-nests sub-timelines into the parent — do **not** manually add them
- Duration comes from `tl.duration()` — do **not** add `data-duration` on composition elements
- Timelines must be finite (no infinite loops or repeats)
- The timeline ID must exactly match the `data-composition-id` attribute on the root element

For a complete guide to working with GSAP timelines, see [GSAP Animation](/guides/gsap-animation).

## Caption Discoverability

For caption compositions, add these attributes to the root node so the framework can identify and special-case caption rendering:

```html
<div
  data-composition-id="captions"
  data-timeline-role="captions"
  data-caption-root="true"
  ...
>
```

## Output Checklist

<Check>
  Before rendering, verify your composition meets these requirements:

  - Every composition has `data-width` and `data-height` on the root element
  - Each reusable composition is in its own HTML file
  - External compositions are loaded via `data-composition-src`
  - Each external composition file uses a `<template>` wrapper
  - All GSAP timelines are registered in `window.__timelines` with the correct ID
  - Timed visible elements (images, divs) have `class="clip"`
  - Video elements do **not** have `class="clip"` (framework manages them directly)
  - All `data-start` references point to existing clip IDs
  - Run `npx hyperframes lint` to catch structural issues automatically
</Check>

## /concepts/compositions

---
title: Compositions
description: "The fundamental building block of a Hyperframes video."
---

A composition is an HTML document that defines a video timeline. Every clip -- video, image, audio -- lives inside a composition.

## Structure

Every composition needs a root element with `data-composition-id`:

```html index.html
<div id="root" data-composition-id="root"
     data-start="0" data-width="1920" data-height="1080">
  <!-- Elements go here -->
</div>
```

The `index.html` file is the top-level composition. It can contain nested compositions within it. Any composition can be imported into another -- there is no special "root" type.

## Clip Types

A clip is any discrete block on the timeline, represented as an HTML element with [data attributes](/concepts/data-attributes):

- `<video>` -- Video clips, B-roll, A-roll
- `<img>` -- Static images, overlays
- `<audio>` -- Music, sound effects
- `<div data-composition-id="...">` -- Nested compositions (animations, grouped sequences)

See the [HTML Schema Reference](/reference/html-schema) for the full list of attributes on each clip type.

## Nested Compositions

You can embed one composition inside another in two ways: loading from an external file or defining it inline. External files are the recommended approach for reusable compositions.

<Tabs>
  <Tab title="External file">
    Reference another HTML file with `data-composition-src`. The framework automatically fetches the file, extracts the `<template>` content, mounts it, executes scripts, and registers the timeline.

    ```html index.html
    <div
      id="el-5"
      data-composition-id="intro-anim"
      data-composition-src="compositions/intro-anim.html"
      data-start="0"
      data-track-index="3"
    ></div>
    ```

    Each external composition file wraps its content in a `<template>` tag:

    ```html compositions/intro-anim.html
    <template id="intro-anim-template">
      <div data-composition-id="intro-anim" data-width="1920" data-height="1080">
        <div class="title">Welcome!</div>

        <style>
          [data-composition-id="intro-anim"] .title {
            font-size: 72px; color: white; text-align: center;
          }
        </style>

        <script>
          const tl = gsap.timeline({ paused: true });
          tl.from(".title", { opacity: 0, y: -50, duration: 1 });
          window.__timelines["intro-anim"] = tl;
        </script>
      </div>
    </template>
    ```
  </Tab>
  <Tab title="Inline">
    Define a nested composition directly inside the parent. This is simpler for one-off compositions that do not need to be reused.

    ```html index.html
    <div id="root" data-composition-id="root"
         data-start="0" data-width="1920" data-height="1080">

      <!-- Inline nested composition -->
      <div id="el-5" data-composition-id="intro-anim"
           data-start="0" data-track-index="3"
           data-width="1920" data-height="1080">
        <div class="title">Welcome!</div>
      </div>

      <script>
        // Timeline for the inline composition
        const introTl = gsap.timeline({ paused: true });
        introTl.from(".title", { opacity: 0, y: -50, duration: 1 });
        window.__timelines["intro-anim"] = introTl;
      </script>
    </div>
    ```

    Inline compositions do not use `<template>` tags or `data-composition-src`.
  </Tab>
</Tabs>

### Project Structure

<Tree>
  <Tree.Folder name="project" defaultOpen>
    <Tree.File name="index.html" />
    <Tree.Folder name="compositions" defaultOpen>
      <Tree.File name="intro-anim.html" />
      <Tree.File name="caption-overlay.html" />
      <Tree.File name="outro-title.html" />
    </Tree.Folder>
    <Tree.Folder name="assets">
      <Tree.File name="video.mp4" />
      <Tree.File name="music.mp3" />
      <Tree.File name="logo.png" />
    </Tree.Folder>
  </Tree.Folder>
</Tree>

## Two Layers: Primitives and Scripts

Every composition has two layers:

- **HTML** -- primitive clips (`video`, `img`, `audio`, nested compositions). The declarative structure: what plays, when, and on which track. Controlled by [data attributes](/concepts/data-attributes).
- **Script** -- effects, transitions, dynamic DOM, canvas, SVG -- creative animation via [GSAP](/guides/gsap-animation). Scripts do **not** control media playback or clip visibility.

<Warning>
  Never use scripts to play/pause/seek media elements or to show/hide clips based on timing. The framework handles this automatically from data attributes. Scripts that duplicate this behavior will conflict with the framework. See [Common Mistakes](/guides/common-mistakes) for examples.
</Warning>

## Variables

HyperFrames does not automatically bind `data-var-*` attributes into your composition DOM or CSS.

The supported pattern is:

1. Declare the variables once on the sub-comp's `<html>` root with `data-composition-variables` (id + type + default).
2. Pass per-instance values on each composition host with `data-variable-values`.
3. Read the resolved values inside the composition with `window.__hyperframes.getVariables()`. The runtime layers the host's `data-variable-values` over the declared defaults on a per-instance basis, so the same source can be embedded multiple times with different values.

```html index.html
<div
  data-composition-id="card-pro"
  data-composition-src="compositions/card.html"
  data-start="0"
  data-track-index="1"
  data-variable-values='{"title":"Pro","color":"#ff4d4f"}'
></div>
<div
  data-composition-id="card-enterprise"
  data-composition-src="compositions/card.html"
  data-start="card-pro"
  data-track-index="1"
  data-variable-values='{"title":"Enterprise","color":"#22c55e"}'
></div>
```

```html compositions/card.html
<html data-composition-variables='[
  {"id":"title","type":"string","label":"Title","default":"Fallback"},
  {"id":"color","type":"color","label":"Color","default":"#111827"}
]'>
  <body>
    <div data-composition-id="card" data-width="1920" data-height="1080">
      <h1 class="title"></h1>

      <style>
        [data-composition-id="card"] {
          --card-color: #111827;
        }

        [data-composition-id="card"] .title {
          color: var(--card-color);
        }
      </style>

      <script>
        // Inside a sub-comp script, getVariables() returns the per-instance
        // values: declared defaults < host data-variable-values overrides.
        const { title, color } = __hyperframes.getVariables();
        const root = document.querySelector('[data-composition-id="card"]');
        root.querySelector(".title").textContent = title;
        root.style.setProperty("--card-color", color);
      </script>
    </div>
  </body>
</html>
```

If you are building tooling on top of `@hyperframes/core`, the same `data-composition-variables` array is readable via `extractCompositionMetadata()` for Studio editing UI and analysis pipelines.

## Listing Compositions

Use the [CLI](/packages/cli) to see all compositions in a project:

```bash
npx hyperframes compositions
```

## Next Steps

<CardGroup cols={2}>
  <Card title="Data Attributes" icon="code" href="/concepts/data-attributes">
    Full reference for timing, media, and composition attributes
  </Card>
  <Card title="GSAP Animation" icon="wand-magic-sparkles" href="/guides/gsap-animation">
    Add animations to your compositions with GSAP timelines
  </Card>
  <Card title="Examples" icon="grid-2" href="/examples">
    Start from built-in examples for common video patterns
  </Card>
  <Card title="HTML Schema Reference" icon="book" href="/reference/html-schema">
    Complete schema for authoring compositions
  </Card>
</CardGroup>

## /guides/common-mistakes

---
title: Common Mistakes
description: "Pitfalls that break Hyperframes compositions."
---

These are mistakes that cannot be caught by the linter. For automated checks, run `npx hyperframes lint` (see [CLI](/packages/cli#lint)).

<Warning>
  The first two mistakes — animating video element dimensions and controlling media playback in scripts — are the most common causes of broken compositions. If your video looks wrong, check these first.
</Warning>

<AccordionGroup>
  <Accordion title="Animating video element dimensions">
    **Symptom:** Video frames stop updating, or browser performance drops severely.

    **Cause:** GSAP animating `width`, `height`, `top`, `left` directly on a `<video>` element can cause the browser to stop rendering frames.

    **Before (broken):**

    ```javascript index.html
    // Animating the video element directly — causes frame rendering to stop
    tl.to("#el-video", { width: 500, height: 280, top: 700, left: 1400 }, 26);
    ```

    **After (fixed):**

    ```html index.html
    <!-- Wrap the video in a div and animate the wrapper -->
    <div id="pip-wrapper" style="position: absolute; width: 1920px; height: 1080px;">
      <video id="el-video" data-start="0" data-track-index="0"
             src="./assets/video.mp4" style="width: 100%; height: 100%;"></video>
    </div>
    ```

    ```javascript index.html
    // Animate the wrapper — the video fills it at 100%
    tl.to("#pip-wrapper", { width: 500, height: 280, top: 700, left: 1400 }, 26);
    ```

    Use a non-timed wrapper `<div>` for visual effects like picture-in-picture. Animate the wrapper; let the video fill it via CSS.
  </Accordion>

  <Accordion title="Controlling media playback in scripts">
    **Symptom:** Audio/video playback is out of sync, or plays when it should not.

    **Cause:** Calling `video.play()`, `video.pause()`, or setting `audio.currentTime` in your scripts. The [framework owns all media playback](/reference/html-schema#framework-managed-behavior).

    **Before (broken):**

    ```javascript index.html
    // Conflicts with framework media sync
    document.getElementById("el-video").play();
    document.getElementById("el-audio").currentTime = 5;
    ```

    **After (fixed):**

    ```javascript index.html
    // Don't control media playback at all. The framework handles it.
    // Use GSAP for visual animations only:
    tl.to("#el-video", { opacity: 1, duration: 0.5 }, 0);
    ```

    The framework reads [`data-start`](/concepts/data-attributes#timing-attributes), [`data-media-start`](/concepts/data-attributes#media-attributes), and [`data-volume`](/concepts/data-attributes#media-attributes) to control when and how media plays. See [Compositions: Two Layers](/concepts/compositions#two-layers-primitives-and-scripts) for the separation between HTML primitives and scripts.
  </Accordion>

  <Accordion title="Composition duration shorter than video">
    **Symptom:** Video plays for a few seconds then stops. Timeline shows 8-10 seconds even though the video is minutes long.

    **Cause:** The composition duration equals the [GSAP timeline duration](/guides/gsap-animation#timeline-duration-and-composition-duration), not `data-duration` on the video. If your last GSAP animation ends at 8 seconds, the composition is 8 seconds long — regardless of how long the video source is.

    **Before (broken):**

    ```javascript index.html
    // Timeline is only 7.8s long — video cuts off after 7.8 seconds
    tl.to("#lower-third", { left: -640, duration: 0.6 }, 7.2);
    ```

    **After (fixed):**

    ```javascript index.html
    tl.to("#lower-third", { left: -640, duration: 0.6 }, 7.2);

    // Extend the timeline to 283 seconds to match the video length
    tl.set({}, {}, 283);
    ```

    `tl.set({}, {}, TIME)` adds a zero-duration tween at the specified time, extending the timeline without affecting any elements.

    <Tip>
      A quick check: run `npx hyperframes compositions` to see the resolved duration of each composition. If it is shorter than expected, your timeline needs extending.
    </Tip>
  </Accordion>

  <Accordion title="Missing class='clip' on timed elements">
    **Symptom:** Elements are always visible, ignoring their `data-start` and `data-duration` timing.

    **Cause:** The [`class="clip"`](/concepts/data-attributes#element-visibility) attribute tells the runtime to manage the element's visibility lifecycle. Without it, the element is always rendered.

    **Before (broken):**

    ```html index.html
    <!-- Missing class="clip" — this element is always visible -->
    <h1 id="title" data-start="2" data-duration="5" data-track-index="0">
      Hello World
    </h1>
    ```

    **After (fixed):**

    ```html index.html
    <!-- With class="clip", the runtime shows this only from 2s to 7s -->
    <h1 id="title" class="clip" data-start="2" data-duration="5" data-track-index="0">
      Hello World
    </h1>
    ```

    <Note>
      The linter catches this one: `npx hyperframes lint` will flag timed elements missing `class="clip"`.
    </Note>
  </Accordion>

  <Accordion title="Oversized source images">
    **Symptom:** Preview stutters during scenes with images on screen. Render is slower than expected.

    **Cause:** Source images at much higher resolution than the canvas. Chrome decodes images to raw RGBA bitmaps before displaying them, and bitmap size is `width × height × 4` bytes — independent of file size on disk. A 7000×5000 JPEG is 140MB decoded, even if the file is only 2MB.

    Displaying such an image in a 384×1080 region wastes memory and forces the compositor to resample a huge texture every frame.

    **Before (bloated):**

    ```html index.html
    <!-- 7000x5000 source, ~140MB decoded -->
    <img class="clip" data-start="0" data-duration="3"
         src="./assets/hero-scene.jpg" />
    ```

    **After (sized to the canvas):**

    ```bash Terminal
    # Resize a batch of images to fit within 3840x3840, preserving aspect ratio
    mkdir -p assets/resized
    mogrify -path assets/resized -resize 3840x3840\> assets/*.jpg
    ```

    ```html index.html
    <!-- ~3840x2560 source, ~40MB decoded -->
    <img class="clip" data-start="0" data-duration="3"
         src="./assets/resized/hero-scene.jpg" />
    ```

    **Rule of thumb:** source images at most 2x the canvas dimensions. For a 1920×1080 composition, 3840×2160 is already plenty. See [Performance: Image sizing](/guides/performance#image-sizing).
  </Accordion>

  <Accordion title="Heavy backdrop-filter stacks">
    **Symptom:** Specific scenes drop to 5-10fps in preview. The composition is fine elsewhere.

    **Cause:** `backdrop-filter: blur()` on large elements, especially stacked at high radii. Each blur layer forces the compositor to sample pixels behind the element, run a blur kernel, and composite the result. Stacked layers multiply the cost.

    **Before (expensive):**

    ```css
    /* 8 layers per side = 16 blur passes every frame */
    .pb-1 { backdrop-filter: blur(1px); }
    .pb-2 { backdrop-filter: blur(2px); }
    .pb-3 { backdrop-filter: blur(4px); }
    .pb-4 { backdrop-filter: blur(8px); }
    .pb-5 { backdrop-filter: blur(16px); }
    .pb-6 { backdrop-filter: blur(32px); }
    .pb-7 { backdrop-filter: blur(64px); }
    .pb-8 { backdrop-filter: blur(128px); }
    ```

    **After (3 tuned layers):**

    ```css
    /* Fewer passes with hand-picked radii — visually similar, much cheaper */
    .pb-1 { backdrop-filter: blur(4px); }
    .pb-2 { backdrop-filter: blur(16px); }
    .pb-3 { backdrop-filter: blur(48px); }
    ```

    **Guidelines:**

    - Keep stacked `backdrop-filter` layers to 2-3 per region
    - Avoid radii above 64px over large areas — the biggest radii dominate the total cost
    - For a static blur effect, pre-render it into a PNG once and overlay with a regular `<img>`

    See [Performance: backdrop-filter: blur()](/guides/performance#backdrop-filter-blur) for the full breakdown.
  </Accordion>

  <Accordion title="Expected HDR output but got SDR">
    **Symptom:** Expected an HDR render, but the output looks the same as SDR or `ffprobe` reports `color_transfer=bt709`.

    **Cause:** By default, Hyperframes only switches to HDR encoding when a source `<video>` or `<img>` is tagged with BT.2020 / PQ / HLG color metadata. Common reasons HDR is not engaged:

    1. **All sources are SDR.** Auto-detect leaves SDR-only compositions in SDR. Verify with `ffprobe`:

       ```bash Terminal
       ffprobe -v error -show_streams source.mp4 | grep color_transfer
       # Want: smpte2084 (PQ) or arib-std-b67 (HLG)
       # SDR:  bt709, smpte170m, bt470bg, etc.
       ```

    2. **Wrong output format.** HDR output requires MP4. `--format mov` and `--format webm` fall back to SDR — Hyperframes logs a warning when this happens.

    3. **SDR was forced.** `--sdr` disables HDR even when HDR sources are present.

    If you need HDR regardless of source metadata, use `--hdr` to force it.

    `--docker` works the same as local rendering — auto-detect, `--hdr`, and `--sdr` are all forwarded into the container and produce the same output decisions (slower, since the container falls back to software WebGL for SDR DOM capture).

    See [HDR Rendering](/guides/hdr) for the full source requirements and verification steps.
  </Accordion>

  <Accordion title="Timeline key doesn't match data-composition-id">
    **Symptom:** Animations don't play. The composition appears static.

    **Cause:** The key used in `window.__timelines` must exactly match the [`data-composition-id`](/concepts/data-attributes#composition-attributes) attribute on the composition root element.

    **Before (broken):**

    ```javascript index.html
    // Mismatch: HTML says "my-video", script registers "root"
    // <div data-composition-id="my-video" ...>
    window.__timelines["root"] = tl;
    ```

    **After (fixed):**

    ```javascript index.html
    // Key matches the data-composition-id attribute
    // <div data-composition-id="my-video" ...>
    window.__timelines["my-video"] = tl;
    ```
  </Accordion>
</AccordionGroup>

## Debugging Checklist

When something does not work, check in this order:

1. **Run the linter:** `npx hyperframes lint` — catches most structural issues
2. **Timeline registered?** Is `window.__timelines["<id>"]` set? Does the key match [`data-composition-id`](/concepts/data-attributes#composition-attributes)?
3. **GSAP-only animations?** Only animate visual properties (opacity, transform, color) — see [GSAP Animation](/guides/gsap-animation#key-rules)
4. **Timeline long enough?** Add `tl.set({}, {}, DURATION)` at the end — see [Timeline Duration](/guides/gsap-animation#timeline-duration-and-composition-duration)
5. **Console errors?** Open browser console — runtime errors show as `[Browser:ERROR]`
6. **Still stuck?** See [Troubleshooting](/guides/troubleshooting) for environment and rendering issues

## Next Steps

<CardGroup cols={2}>
  <Card title="Troubleshooting" icon="wrench" href="/guides/troubleshooting">
    Fix environment and rendering issues
  </Card>
  <Card title="GSAP Animation" icon="wand-magic-sparkles" href="/guides/gsap-animation">
    Review animation rules and patterns
  </Card>
  <Card title="HTML Schema Reference" icon="code" href="/reference/html-schema">
    Full attribute reference and checklist
  </Card>
  <Card title="Data Attributes" icon="database" href="/concepts/data-attributes">
    Timing, media, and composition attributes
  </Card>
</CardGroup>

## /guides/gsap-animation

---
title: GSAP Animation
description: "Add animations to your Hyperframes compositions with GSAP."
---

Hyperframes uses [GSAP](https://gsap.com/) for animation. Timelines are paused and controlled by the runtime — you define the animations, the framework handles playback. For background on how animation runtimes plug into Hyperframes, see [Frame Adapters](/concepts/frame-adapters).

## Setup

Include GSAP and create a paused timeline:

```html index.html
<script src="https://cdn.jsdelivr.net/npm/gsap@3/dist/gsap.min.js"></script>
<script>
  // 1. Create a paused timeline — the framework controls playback
  const tl = gsap.timeline({ paused: true });

  // 2. Add animations using the position parameter (3rd arg) for absolute timing
  tl.to("#title", { opacity: 1, duration: 0.5 }, 0);

  // 3. Initialize the global timelines registry (if not already present)
  window.__timelines = window.__timelines || {};

  // 4. Register the timeline using the data-composition-id as the key
  window.__timelines["root"] = tl;
</script>
```

<Note>
  The key you use in `window.__timelines` must match the `data-composition-id` attribute on your composition's root element. See [Compositions](/concepts/compositions) for how the root element is structured.
</Note>

## Key Rules

1. **Always create timelines with `{ paused: true }`** — the framework controls playback via [deterministic seeking](/concepts/determinism)
2. **Register timelines on `window.__timelines`** with the [`data-composition-id`](/concepts/data-attributes#composition-attributes) as key
3. **Use the position parameter** (3rd argument) for absolute timing: `tl.to(el, vars, 1.5)`
4. **Only animate visual properties** — never control media playback in scripts

## Supported Methods

| Method | Description |
|--------|-------------|
| `tl.to(target, vars, position)` | Animate to values |
| `tl.from(target, vars, position)` | Animate from values |
| `tl.fromTo(target, fromVars, toVars, position)` | Animate from/to values |
| `tl.set(target, vars, position)` | Set values instantly |

## Supported Properties

`opacity`, `x`, `y`, `scale`, `scaleX`, `scaleY`, `rotation`, `width`, `height`, `visibility`, `color`, `backgroundColor`, and any CSS-animatable property.

## Timeline Duration and Composition Duration

A composition's duration equals its GSAP timeline duration. The two are directly linked:

```javascript compositions/intro-anim.html
// Your last animation ends at 3 seconds...
tl.from("#title", { opacity: 0, y: -50, duration: 1 }, 0);
tl.to("#title", { opacity: 0, duration: 1 }, 2);
// ...so this composition is exactly 3 seconds long.
```

If your composition contains a video clip that is 283 seconds long, but your last GSAP animation ends at 8 seconds, the composition will be only 8 seconds long and the video will be cut short. To extend the timeline to match the video:

```javascript index.html
// All your visual animations
tl.to("#lower-third", { left: -640, duration: 0.6 }, 7.2);

// Extend the timeline to 283 seconds to match the video length.
// This adds a zero-duration tween at 283s without affecting any elements.
tl.set({}, {}, 283);
```

<Warning>
  This is one of the most common mistakes in Hyperframes. If your video cuts off early, the timeline is too short. See [Common Mistakes: Composition Duration Shorter Than Video](/guides/common-mistakes) for more details.
</Warning>

## What NOT to Do

These patterns will break your composition or cause sync issues:

```javascript index.html
// WRONG: Playing media in scripts — the framework owns media playback
document.getElementById("el-video").play();
document.getElementById("el-audio").currentTime = 5;

// WRONG: Creating a non-paused timeline
const tl = gsap.timeline(); // missing { paused: true }!

// WRONG: Animating dimensions directly on a <video> element
tl.to("#el-video", { width: 500, height: 280 }, 5);

// WRONG: Manually nesting sub-timelines
const masterTL = window.__timelines["root"];
masterTL.add(window.__timelines["intro-anim"], 0);
```

The framework automatically manages [media playback](/reference/html-schema#framework-managed-behavior), [clip lifecycle](/concepts/compositions#two-layers-primitives-and-scripts), and [sub-composition nesting](#sub-composition-timelines). Scripts that duplicate this behavior will conflict.

## Sub-Composition Timelines

Each [nested composition](/concepts/compositions#nested-compositions) registers its own timeline. The framework automatically nests sub-composition timelines into the parent based on [`data-start`](/concepts/data-attributes#timing-attributes):

```javascript compositions/intro-anim.html
// In compositions/intro-anim.html
const tl = gsap.timeline({ paused: true });
tl.from(".title", { opacity: 0, y: -50, duration: 1 });
window.__timelines["intro-anim"] = tl;

// DO NOT manually add sub-timelines to the master:
// masterTL.add(window.__timelines["intro-anim"], 0); // UNNECESSARY
```

<Warning>
  Don't animate `width`, `height`, `top`, or `left` directly on `<video>` elements — this can cause the browser to stop rendering frames. Wrap the video in a `<div>` and animate the wrapper instead. See [Common Mistakes](/guides/common-mistakes) for a detailed explanation.
</Warning>

## Next Steps

<CardGroup cols={2}>
  <Card title="Compositions" icon="layer-group" href="/concepts/compositions">
    Understand the building blocks that timelines animate
  </Card>
  <Card title="Frame Adapters" icon="plug" href="/concepts/frame-adapters">
    Learn how GSAP plugs into the render pipeline
  </Card>
  <Card title="Common Mistakes" icon="triangle-exclamation" href="/guides/common-mistakes">
    Avoid pitfalls that break animations
  </Card>
  <Card title="HTML Schema Reference" icon="code" href="/reference/html-schema">
    Full reference for composition attributes
  </Card>
</CardGroup>

## /guides/prompting

---
title: Prompt Guide
description: "How to prompt Claude Code, Cursor, Codex, Google Antigravity, GitHub Copilot CLI, and other AI agents to author Hyperframes compositions — with copy-pasteable examples and vocabulary tables."
---

Hyperframes is built for AI agents — compositions are plain HTML, the CLI is non-interactive, and the framework ships [skills](https://github.com/vercel-labs/skills) that teach agents the patterns docs alone don't cover. This guide shows how to prompt agents effectively once skills are installed — the vocabulary that changes output, the iteration patterns that save time, and the rules that prevent breakage.

## One-time setup

Install the skills in your project (or globally for your agent):

```bash
npx skills add heygen-com/hyperframes
```

In Claude Code, restart the session after installing. Skills register as **slash commands**:

| Slash command              | What it loads                                                              |
| -------------------------- | -------------------------------------------------------------------------- |
| `/hyperframes`             | Composition authoring — HTML structure, timing, captions, TTS, transitions |
| `/hyperframes-cli`         | Dev-loop CLI — `init`, `lint`, `inspect`, `preview`, `render`, `doctor`    |
| `/hyperframes-media`       | Asset preprocessing — `tts`, `transcribe`, `remove-background`             |
| `/hyperframes-registry`    | Block and component installation via `hyperframes add`                     |
| `/website-to-hyperframes`  | Capture a URL and turn it into a video; runs the full [Hyperframes pipeline](/guides/pipeline) |
| `/gsap`                    | GSAP animation API — timelines, easing, ScrollTrigger, plugins             |

<Tip>
  Always prefix Hyperframes prompts with `/hyperframes` (or invoke the skill another way for non-Claude agents). This loads the skill context explicitly so the agent gets composition rules right the first time, instead of relying on whatever it remembers about web video.
</Tip>

## Claude Design

Claude Design uses a different setup. Download [`claude-design-hyperframes.md`](https://github.com/heygen-com/hyperframes/blob/main/docs/guides/claude-design-hyperframes.md) from GitHub (click the ↓ button), then **attach it to your chat** (don't paste the URL — file attachments produce better output):

```text
Use the attached skill. 25-second LinkedIn video for my startup.

Problem: Sales teams waste 3 hours/day on manual CRM updates.
Solution: AutoCRM — AI that logs every call, email, and meeting.
Traction: 200+ teams, $1.2M ARR, 18% MoM growth.
CTA: autocrmhq.com
```

Claude Design produces a valid first draft (brand identity, scene content, animations, transitions). Download the ZIP and refine in any AI coding agent with `npx hyperframes preview` running. See the [Claude Design guide](/guides/claude-design) for the full workflow.

## The two prompt shapes

Most successful Hyperframes prompts fall into one of two shapes.

### Cold start — describe the video

You tell the agent what you want from scratch. Best for greenfield work where you have the creative direction in your head.

> Using `/hyperframes`, create a 10-second product intro with a fade-in title over a dark background and subtle background music.

> Make a 9:16 TikTok-style hook video about [topic] using `/hyperframes`, with bouncy captions synced to a TTS narration.

Cold-start prompts work best when you specify:

- **Duration** (e.g. "10 seconds", "30s", "5 scenes of 3s each")
- **Aspect ratio** ("16:9", "9:16 vertical", "1:1 square") — defaults to 1920x1080 otherwise
- **Mood / style** ("minimal Swiss grid", "warm grain analog", "high-energy social")
- **Key elements** (title, lower third, captions, background video, music)

### Warm start — turn context into a video

You give the agent something to work with — a URL, a doc, a CSV, a transcript — and ask it to synthesize that into a video. This is where Hyperframes shines because the agent does the research/summarization step *and* the production step in one flow.

> Take a look at this GitHub repo https://github.com/heygen-com/hyperframes and explain its uses and architecture to me using `/hyperframes`.

> Summarize the attached PDF into a 45-second pitch video using `/hyperframes`.

> Read this changelog and turn the top three changes into a 30-second release announcement video using `/hyperframes`.

> Turn this CSV into an animated bar chart race using `/hyperframes`.

Warm-start prompts produce richer, more grounded videos because the agent is writing about *something specific* instead of inventing copy.

## Iterating

Hyperframes is a conversation. After the first render, talk to the agent the way you'd talk to a video editor — don't re-prompt from scratch:

> Make the title 2x bigger.

> Swap to dark mode.

> Add a fade-out at the end and a lower third at 0:03 with my name and title.

> The captions are too small and they overlap the lower third. Move them up and shrink them.

> Replace the background music with `assets/track.mp3`.

The agent already has the composition open and the skills loaded — small targeted edits produce better results than long re-specifications.

## Vocabulary that changes output

The skills map natural-language adjectives to specific framework settings. Using the right word gets you the right result without specifying technical details.

### Motion & easing

Describe how motion should *feel* and the agent picks the matching GSAP ease:

| Say this    | Agent uses       | Feels like                     |
| ----------- | ---------------- | ------------------------------ |
| smooth      | `power2.out`     | Natural deceleration           |
| snappy      | `power4.out`     | Quick and decisive             |
| bouncy      | `back.out`       | Overshoots then settles        |
| springy     | `elastic.out`    | Oscillates into place          |
| dramatic    | `expo.out`       | Fast start, long glide         |
| dreamy      | `sine.inOut`     | Slow, symmetrical              |

**Timing shorthand:** fast (0.2s) = energy, medium (0.4s) = professional, slow (0.6s) = luxury, very slow (1–2s) = cinematic.

### Caption tones

Describe the *energy* of your captions and the agent picks matching typography, size, and animation:

| Tone         | Typography             | Animation    | Size range |
| ------------ | ---------------------- | ------------ | ---------- |
| Hype         | Heavy weight fonts     | Scale-pop    | 72–96px    |
| Corporate    | Clean sans-serif       | Fade + slide | 56–72px    |
| Tutorial     | Monospace              | Typewriter   | 48–64px    |
| Storytelling | Serif                  | Slow fade    | 44–56px    |
| Social       | Rounded, playful       | Bounce       | 56–80px    |

```
"Hype-style captions with scale-pop"
"Calm, elegant subtitles with slow fades"
"Karaoke-style word highlighting"
```

Per-word styling also works:

```
"Make brand names larger with accent color"
"Add bounce to emotional keywords"
"Highlight numbers differently"
```

### Transitions

Every multi-scene composition benefits from transitions. Describe the energy level:

| Energy  | CSS option       | Shader option       |
| ------- | ---------------- | ------------------- |
| Calm    | Blur crossfade   | Cross-warp morph    |
| Medium  | Push slide       | Whip pan            |
| High    | Zoom through     | Glitch, ridged burn |

Or describe by mood:

```
"Warm transitions for this wellness brand"
"Cold, clinical transitions for tech"
"Playful bouncy transitions"
"Dramatic zoom for the reveal"
```

### Audio-reactive animation

Map audio frequency bands to visual properties. The agent uses these defaults:

| Audio band | Maps to   | Visual effect       |
| ---------- | --------- | ------------------- |
| Bass       | `scale`   | Pulse on the beat   |
| Treble     | `glow`    | Shimmer intensity   |
| Amplitude  | `opacity` | Breathing           |
| Mids       | `shape`   | Morphing            |

```
"Make the text pulse with the beat"
"Add bass-driven scale to the logo"
"Create glow that responds to treble"
```

<Tip>
  Keep audio-reactive effects subtle for text (3–6% intensity). Go bigger for backgrounds (10–30%).
</Tip>

### Marker highlights

Hand-drawn emphasis effects for text:

| Mode        | Effect             | Best for      |
| ----------- | ------------------ | ------------- |
| `highlight` | Marker sweep       | Key phrases   |
| `circle`    | Hand-drawn ellipse | Single words  |
| `burst`     | Radiating lines    | Hype moments  |
| `scribble`  | Chaotic scratch    | Crossing out  |
| `sketchout` | Rectangle outline  | Callouts      |

```
"Add a marker highlight sweep on 'revolutionary'"
"Circle this keyword with hand-drawn effect"
"Add burst lines around 'AMAZING'"
```

### Text-to-speech voices

TTS runs locally via Kokoro (no API key needed). Describe the content and the agent picks a voice, or request one directly:

| Content type  | Recommended voices         |
| ------------- | -------------------------- |
| Product demo  | `af_heart`, `af_nova`      |
| Tutorial      | `am_adam`, `bf_emma`       |
| Marketing     | `af_sky`, `am_michael`     |

```
"Generate narration for this script"
"Create voiceover with a professional female voice"
"Add TTS with British male voice at 1.1x speed"
```

### Rendering quality

| Quality    | Use for                  |
| ---------- | ------------------------ |
| `draft`    | Fast iteration           |
| `standard` | Review and feedback      |
| `high`     | Final delivery           |

```
"Quick draft render"
"Render at high quality"
"Export as transparent WebM"
```

## Rules to know

The skills enforce these automatically, but if you hand-edit compositions or debug issues, these are the rules that matter:

1. **Register all timelines** on `window.__timelines` — the renderer can't seek animations it doesn't know about.
2. **Video elements must be `muted`** — audio goes in separate `<audio>` elements so the renderer can mix it.
3. **No `Math.random()`** — random values produce different frames on each render, breaking determinism. Use a seeded PRNG (e.g. mulberry32) if you need pseudo-random values.
4. **Synchronous timeline construction** — no `async`/`await` or `fetch()` during GSAP timeline setup.
5. **Timed elements need `class="clip"`** — plus `data-start`, `data-duration`, and `data-track-index`.
6. **Add entrance animations to every scene** — elements appearing without animation feel broken on video.
7. **Add transitions between scenes** — jump cuts between scenes are almost always unintentional in composed video.

<Warning>
  Rules 1–5 are technical requirements — breaking them produces incorrect renders. Rules 6–7 are best practices that the skills apply by default. You can override them when you have a reason to.
</Warning>

## Anti-patterns

Things that cause friction (or wrong output):

- **Don't ask for React / Vue components.** Hyperframes compositions are plain HTML with `data-*` attributes and a GSAP timeline. Asking for "a React component for the intro" forces the agent to translate later.
- **Don't ask for 4K or 60fps unless you need it.** Defaults (1920×1080, 30fps) render fast and look great. Higher specs slow rendering meaningfully.
- **Don't skip the slash command.** Without `/hyperframes`, the agent may guess at HTML video conventions instead of using the framework's actual rules (`class="clip"` on timed elements, `window.__timelines` registration, etc.).
- **Don't paste long error logs into the prompt without context.** Run `npx hyperframes lint` and `npx hyperframes validate` first — lint catches structural issues, validate catches runtime errors (JS exceptions, missing assets, contrast problems).
- **Don't assume the agent knows your assets.** Mention file paths explicitly (`assets/intro.mp4`, `assets/logo.png`) — the agent will check what's there but a hint speeds it up.

## Recommended workflow

1. `npx hyperframes init my-video` — scaffold a project (skills install automatically)
2. Open the project in Claude Code (or Cursor / Codex)
3. Prompt with `/hyperframes` and one of the shapes above
4. `npx hyperframes preview` — watch in the browser as the agent edits
5. Iterate with small targeted prompts
6. `npx hyperframes render --output final.mp4` when you're happy

## Next steps

<CardGroup cols={2}>
  <Card title="Quickstart" icon="rocket" href="/quickstart">
    Build and render your first video
  </Card>
  <Card title="Common Mistakes" icon="circle-exclamation" href="/guides/common-mistakes">
    Pitfalls the linter can't catch
  </Card>
  <Card title="GSAP Animation" icon="wand-magic-sparkles" href="/guides/gsap-animation">
    Add fade, slide, scale, and custom animations
  </Card>
  <Card title="Catalog" icon="grid-2" href="/catalog/blocks/data-chart">
    50+ ready-to-use blocks and components
  </Card>
</CardGroup>

## OUTPUT CONTRACT

Return exactly one complete self-contained HTML document.
Do not include explanations, notes, or markdown code fences.

## Sources

- /home/runner/work/hyperframes/hyperframes/docs/reference/html-schema.mdx
- /home/runner/work/hyperframes/hyperframes/docs/concepts/compositions.mdx
- /home/runner/work/hyperframes/hyperframes/docs/guides/common-mistakes.mdx
- /home/runner/work/hyperframes/hyperframes/docs/guides/gsap-animation.mdx
- /home/runner/work/hyperframes/hyperframes/docs/guides/prompting.mdx

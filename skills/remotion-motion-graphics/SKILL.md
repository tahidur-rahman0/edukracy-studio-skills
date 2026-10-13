---
name: remotion-motion-graphics
description: Design animated titles, kinetic typography, lower thirds, app-screen promos, explainer cards and end cards in Edukracy Studio by writing a Remotion (React) scene for create_motion_graphic. Use whenever the user asks for an intro, title card, animated text, promo, explainer graphic, CTA/end card, or "make it look designed" instead of filmed footage.
---

# Remotion motion graphics in Edukracy Studio

`create_motion_graphic` renders one React scene with Remotion on the user's Mac at the project's resolution and
frame rate. You write the whole scene in one TSX file. The app validates it, renders it, and the result appears in
the media library (and on the timeline if you pass `placeAtFrame`).

## Hard rules

1. **One file, default export.** `export default function Scene() { ... }`. No other files exist.
2. **Allowed imports only:** `react`, `remotion`, `@remotion/google-fonts/<Font>`, `@remotion/transitions` (and its
   subpaths like `@remotion/transitions/fade`), `@remotion/shapes`. No `require`, no dynamic `import()`, no Node APIs,
   no images or video files from disk. Anything else is rejected before rendering.
3. **Everything is a function of the frame.** Use `useCurrentFrame()` and `useVideoConfig()`. Never use
   `Date.now()`, `Math.random()` without a seed, timers, CSS transitions or CSS animations — renders are
   frame-by-frame and those break.
4. **Size from the canvas, not constants.** Read `width`, `height`, `fps`, `durationInFrames` from
   `useVideoConfig()`; the same scene must work at 1920×1080 and 1080×1920.
5. **Poll and fix.** After calling the tool, poll `get_media` with the returned `mediaRef`. On `failed`, read the
   message, fix the code, call `create_motion_graphic` again. Never claim success before status is `none`.

## Building blocks

```tsx
import { AbsoluteFill, Sequence, interpolate, spring, useCurrentFrame, useVideoConfig, Easing } from "remotion";
import { loadFont } from "@remotion/google-fonts/Inter";

const { fontFamily } = loadFont();

export default function Scene() {
  const frame = useCurrentFrame();
  const { fps, width } = useVideoConfig();
  const enter = spring({ frame, fps, config: { damping: 200 } });           // 0 → 1, settles in ~1 s
  const fade = interpolate(frame, [0, 12], [0, 1], { extrapolateRight: "clamp" });
  return (
    <AbsoluteFill style={{ background: "#F4F8FF", justifyContent: "center", alignItems: "center", fontFamily }}>
      <div style={{ opacity: fade, transform: `translateY(${(1 - enter) * 40}px)`,
                    fontSize: width * 0.05, fontWeight: 800, color: "#0E2D5C" }}>
        A study plan that builds itself.
      </div>
      <Sequence from={Math.round(fps * 1)}>
        {/* children start 1 s in; their useCurrentFrame() restarts at 0 */}
      </Sequence>
    </AbsoluteFill>
  );
}
```

- **Entrances:** `spring` for motion, `interpolate` with `extrapolateLeft/Right: "clamp"` for opacity and position.
- **Exits:** compute from the end: `interpolate(frame, [durationInFrames - 12, durationInFrames], [1, 0], clamp)`.
- **Staggering:** offset each item by `i * 4` frames inside one `spring`/`interpolate` call.
- **Timing:** `Sequence from / durationInFrames` for beats; keep each text beat on screen ≥ 1.5 s (≥ 45 frames at 30 fps).
- **Easing:** `Easing.bezier(0.16, 1, 0.3, 1)` for confident UI motion.

## Layout & type

- Safe area: keep text inside 8 % margins of width and height.
- Headline 5–7 % of width (16:9) or 8–10 % (9:16); body ≥ 2.8 % of width; never below 28 px at 1080p.
- Max 2 fonts. Good Google fonts: Inter, Manrope, Poppins, Plus Jakarta Sans; for Hindi/Devanagari use Noto Sans Devanagari.
- Contrast ≥ 4.5:1. Edukracy palette: navy `#0E2D5C`, blue `#2F7DE1`, sky `#BFE4FF`, background `#F4F8FF`, accent orange `#FF9F1A`.

## Overlays (titles over footage)

Pass `transparent: true` and **do not paint a background** (no `background` on AbsoluteFill). Typical lower third:
a rounded bar sliding in from the left with name + role, out in the last 12 frames. Place it with `placeAtFrame`
over the speaker; it lands on a new top track above the footage.

## Recipe: pitch / explainer video (Edukracy style)

1. Ask for or derive 4–6 beats from the script (headline, problem, feature ×2–3, CTA).
2. One scene per beat, 3–6 s each, same font + palette, consistent entrance style.
3. Feature beat = headline left, a phone mock on the right: a rounded rectangle (`borderRadius` ≈ 6 % of its width,
   dark bezel) with simple cards inside built from divs — never screenshots unless the user imports them.
4. End card: logo text "Edukracy", one-line value prop, CTA in the accent color.
5. Place beats back-to-back with `placeAtFrame`, or render them and arrange with `add_clips`; then add voiceover/music
   and captions with the normal editing tools.
6. Verify with `inspect_timeline` on a frame inside each beat.

## Notes

- Long scenes render slower; prefer several 3–8 s scenes over one 60 s scene.
- Fonts load from Google Fonts at render time; if a font fails, fall back to Inter.
- Keep code under ~300 lines; extract small components inside the same file.

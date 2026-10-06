---
name: remotion-best-practices
description: Best practices for building videos in React with Remotion. Use whenever writing, reviewing or debugging Remotion code — compositions, animations (interpolate/spring), Sequence/Series timing, transitions, video/audio/image assets, fonts, captions, data-driven videos (calculateMetadata), and rendering (CLI, Lambda, Player).
---

# Remotion best practices

Remotion renders a React component frame by frame. Every frame must be a pure
function of the frame number and props. Most bugs come from breaking that rule.

## 1. Golden rules

- Drive **all** motion from `useCurrentFrame()` and `useVideoConfig()`.
- **Never** use CSS transitions, CSS `@keyframes`, Tailwind `animate-*` classes,
  `requestAnimationFrame`, `setTimeout`, or `useFrame()` from react-three-fiber
  for animation — they are not tied to the frame and break in renders.
- **Never** use `Math.random()`. Use `random("seed")` from `remotion` so every
  render (and every parallel render worker) gets the same value.
- Express durations in seconds × `fps`, not hard-coded frame numbers:
  `const { fps } = useVideoConfig(); const d = 2 * fps;`
- Keep all `remotion` and `@remotion/*` packages on the **exact same version**
  (no `^`). Check with `npx remotion versions`, upgrade with `npx remotion upgrade`.

## 2. Compositions

Register compositions in `src/Root.tsx`:

```tsx
import { Composition } from "remotion";
import { z } from "zod";
import { zColor } from "@remotion/zod-types";
import { MyVideo } from "./MyVideo";

export const myVideoSchema = z.object({
  title: z.string(),
  color: zColor(),
});

export const RemotionRoot = () => (
  <Composition
    id="MyVideo"
    component={MyVideo}
    durationInFrames={150}
    fps={30}
    width={1920}
    height={1080}
    schema={myVideoSchema}
    defaultProps={{ title: "Hello", color: "#0b84f3" }}
  />
);
```

- Use a zod `schema` so props are typed and editable in the Studio.
- Use `<Still>` for single images (thumbnails, OG images).
- Common sizes: 1920×1080 (16:9), 1080×1920 (9:16 Reels/TikTok/Shorts), 1080×1080.

## 3. Animation

```tsx
import { interpolate, spring, useCurrentFrame, useVideoConfig, Easing } from "remotion";

const frame = useCurrentFrame();
const { fps } = useVideoConfig();

// Linear / eased mapping — always clamp unless you want extrapolation
const opacity = interpolate(frame, [0, 0.5 * fps], [0, 1], {
  extrapolateLeft: "clamp",
  extrapolateRight: "clamp",
  easing: Easing.bezier(0.16, 1, 0.3, 1),
});

// Physics-based 0 → 1
const enter = spring({ frame, fps, config: { damping: 200 } }); // no bounce
const pop = spring({ frame: frame - 10, fps, config: { damping: 12 } }); // delayed, bouncy

const scale = interpolate(enter, [0, 1], [0.8, 1]);
```

- `spring()` returns a progress value; map it to real units with `interpolate()`.
- Delay a spring by passing `frame - delay`. Limit its length with `durationInFrames`.
- Animate `transform` and `opacity`; avoid animating layout properties
  (`width`, `top`) when a transform works.

## 4. Timing and sequencing

- `<AbsoluteFill>` for full-frame layered layouts.
- `<Sequence from={30} durationInFrames={60}>` — inside it, `useCurrentFrame()`
  starts at 0. Build scenes as self-contained components that animate from frame 0.
- `<Series>` / `<Series.Sequence durationInFrames={...} offset={...}>` for
  back-to-back scenes.
- Add `premountFor={fps}` to Sequences that contain media so they load before
  they appear. Use `layout="none"` when you don't want the AbsoluteFill wrapper.
- `<Loop durationInFrames={...}>` for repeating elements, `<Freeze frame={...}>`
  to hold a frame.

### Transitions

```tsx
import { TransitionSeries, linearTiming, springTiming } from "@remotion/transitions";
import { fade } from "@remotion/transitions/fade";
import { slide } from "@remotion/transitions/slide";

<TransitionSeries>
  <TransitionSeries.Sequence durationInFrames={60}><SceneA /></TransitionSeries.Sequence>
  <TransitionSeries.Transition presentation={fade()} timing={linearTiming({ durationInFrames: 15 })} />
  <TransitionSeries.Sequence durationInFrames={60}><SceneB /></TransitionSeries.Sequence>
</TransitionSeries>
```

Transitions overlap scenes, so total duration = sum of sequences − sum of
transitions. Compute `durationInFrames` of the composition accordingly.

## 5. Assets

- Put files in `public/` and reference them with `staticFile("logo.png")`.
  Remote URLs work too.
- Use Remotion's components, which wait for loading before a frame is captured:
  `<Img>`, `<Video>` / `<OffthreadVideo>`, `<Audio>`, `<IFrame>`, `<Gif>` (`@remotion/gif`),
  `<Lottie>` (`@remotion/lottie`). Never a plain `<img>` / `<video>`.
- Media props: `startFrom`/`endAt` (trim, in frames), `volume` (number or
  `(f) => number` for fades), `playbackRate`, `muted`, `loop`.
- Get media length for dynamic durations with `parseMedia()` from
  `@remotion/media-parser` (or `getVideoMetadata`/`getAudioDurationInSeconds`
  from `@remotion/media-utils`) inside `calculateMetadata`.
- Audio visualization: `useAudioData()` + `visualizeAudio()` from `@remotion/media-utils`.

## 6. Fonts

```tsx
import { loadFont } from "@remotion/google-fonts/Inter";
const { fontFamily } = loadFont("normal", { weights: ["400", "700"], subsets: ["latin", "cyrillic"] });
```

- Load only the weights/subsets you need. Include `cyrillic` / `cyrillic-ext`
  for Kazakh or Russian text.
- Local fonts: `loadFont({ family, url: staticFile("font.woff2") })` from `@remotion/fonts`.
- Measure text with `@remotion/layout-utils` (`measureText`, `fitText`) after the font has loaded.

## 7. Data-driven and async work

Prefer `calculateMetadata` on `<Composition>` for fetching data and computing
duration/size before rendering:

```tsx
calculateMetadata={async ({ props, abortSignal }) => {
  const data = await fetch(props.url, { signal: abortSignal }).then((r) => r.json());
  return { durationInFrames: data.items.length * 90, props: { ...props, data } };
}}
```

If you must load inside a component, use `delayRender()` / `continueRender()`
(or `cancelRender(err)` on failure) with `useState(() => delayRender())`.
Every `delayRender` must be resolved within the timeout (default 30 s).

## 8. Captions and text

- Use `@remotion/captions` (`Caption` type, `createTikTokStyleCaptions()`) for
  word-by-word subtitles; transcribe with `@remotion/install-whisper-cpp`
  or an external API.
- Keep text inside safe margins (~5–10 % of the frame), especially for 9:16
  social formats where UI covers the top and bottom.

## 9. 3D, charts, other libraries

- 3D: `<ThreeCanvas>` from `@remotion/three`, animate with `useCurrentFrame()`.
- Charts/SVG: animate values with `interpolate`; disable the library's own
  animations (e.g. `isAnimationActive={false}` in Recharts).
- Any library with its own clock/animations must have them turned off.

## 10. Rendering

```bash
npx remotion studio                                   # preview
npx remotion render MyVideo out/video.mp4             # render
npx remotion render MyVideo out/video.mp4 --props='{"title":"Hi"}'
npx remotion still MyVideo out/thumb.png --frame=30
```

- Codecs: `h264` (default, MP4), `h265`, `vp8`/`vp9` (WebM), `prores`
  (alpha with `--image-format=png --pixel-format=yuva444p10le`), `gif`.
- Speed: tune `--concurrency`; use `<OffthreadVideo>`; avoid heavy CSS
  filters/`box-shadow`/`backdrop-filter`; memoize expensive computations with `useMemo`.
- Serverless: `@remotion/lambda` or `@remotion/cloudrun`. Embed in web apps with
  `<Player>` from `@remotion/player`.

## 11. Review checklist

- [ ] No CSS/Tailwind animations, timers, `Math.random()`.
- [ ] Every `interpolate` that should stop is clamped.
- [ ] Durations derived from `fps`.
- [ ] Assets via `staticFile()` and Remotion components; media Sequences premounted.
- [ ] Fonts loaded with the right subsets.
- [ ] All `@remotion/*` versions identical.
- [ ] Composition duration matches content (incl. transition overlap).
- [ ] Renders correctly with `npx remotion render`, not just in the Studio.

## 12. License note

Remotion is free for individuals and companies with up to 3 employees; larger
companies need a company license (see remotion.dev/license).

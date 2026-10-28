# ClipForge — Technical Specification & Product Requirements

**Version:** 1.0
**Target platform:** Windows 10 / 11, 64-bit
**Deliverable:** A desktop non-linear video editor packaged as a double-clickable `.exe` (installer + portable)
**Audience:** Claude Code (implementer) and the project owner (reviewer)

> **Instructions to Claude Code**
> This document is the single source of truth. Read it fully before writing code.
> Build strictly in the phase order of **Section 20**. Each phase must build, launch from the packaged `.exe`, and pass its acceptance checks before the next phase starts.
> When something is ambiguous, choose the behavior of Adobe Premiere Pro / DaVinci Resolve, and record the decision in `docs/DECISIONS.md`.
> Track progress in `PROGRESS.md` using the feature IDs in Section 15.

---

## Table of contents

1. Product overview
2. Users and core use cases
3. Technology stack
4. System architecture
5. Repository structure
6. Time model
7. Data model
8. Project file format
9. State management and undo/redo
10. IPC contract
11. Media pipeline (import, probe, thumbnails, waveforms, proxies)
12. Compositor (shared rendering core)
13. Preview playback engine
14. Export engine
15. Feature requirements (with IDs and phases)
16. UI specification
17. Keyboard shortcuts
18. Non-functional requirements
19. Error handling, logging, testing
20. Build phases and acceptance criteria
21. Packaging and distribution
22. Working rules for Claude Code
23. Glossary

---

## 1. Product overview

ClipForge is a non-destructive, multi-track video editor. Users import media, arrange it on a timeline, add text, transitions and effects, preview in real time, and export to MP4 and other formats. It targets creators making YouTube videos, shorts/reels, tutorials and vlogs — a desktop experience between CapCut and Premiere Pro.

### Goals
- Launch by double-clicking an `.exe`; no separate installs (FFmpeg is bundled).
- Smooth, frame-accurate editing of 1080p (and 4K via proxies) footage.
- **What you see is what you export**: preview and export use the same compositor code.
- Never modify source media.

### Non-goals (v1)
- macOS / Linux builds
- Cloud sync, accounts, collaboration
- Third-party plugin system
- RAW camera formats, 8K, HDR grading
- Multicam editing

---

## 2. Users and core use cases

| # | Use case |
|---|---|
| UC-1 | Cut a screen recording: remove mistakes, add title and background music, export 1080p for YouTube |
| UC-2 | Make a vertical short (1080×1920) from landscape footage with captions and punchy text |
| UC-3 | Assemble a travel vlog from many phone clips with transitions, music and color correction |
| UC-4 | Picture-in-picture tutorial: webcam overlay on screen recording |
| UC-5 | Talking-head video: auto-remove silences, auto-generate captions |

---

## 3. Technology stack

Do not change these choices without asking the owner.

| Concern | Choice | Notes |
|---|---|---|
| Shell | Electron (latest stable) | Chromium gives WebGL2, Web Audio, HTML media decoding |
| Language | TypeScript, `strict: true` | Everywhere, including scripts |
| Build | `electron-vite` | Separate builds for main, preload, renderer, export window |
| UI | React 18 | Function components + hooks |
| Styling | Tailwind CSS | Dark theme default |
| UI primitives | Radix UI (dialogs, menus, sliders, tooltips) | Accessible and unstyled |
| State | Zustand + Immer (`produceWithPatches`) | Undo/redo via patches (Section 9) |
| Validation | Zod | Project file and IPC payload validation |
| Rendering | WebGL2 via `twgl.js` | Shared compositor (Section 12) |
| Timeline drawing | HTML Canvas 2D | Not DOM-per-clip, for performance |
| Media engine | FFmpeg + FFprobe binaries | Bundled; spawned as child processes |
| Logging | `electron-log` | File logs in `%APPDATA%/ClipForge/logs` |
| Settings | JSON file in `userData` (own small module, Zod-validated) | |
| IDs | `nanoid` | |
| Unit tests | Vitest | |
| E2E tests | Playwright (`_electron`) | |
| Lint/format | ESLint + Prettier | |
| Packaging | electron-builder | NSIS installer + portable `.exe` |

**FFmpeg binaries:** use `ffmpeg-static` and `ffprobe-static` (with `asarUnpack`), or a script `scripts/fetch-ffmpeg.ts` that downloads a Windows x64 build into `resources/bin/`. Paths are resolved by one module (`main/services/ffmpeg/paths.ts`) that works in dev and packaged builds (`process.resourcesPath`).

No dependencies requiring `node-gyp`/native compilation unless unavoidable (and then justified in `docs/DECISIONS.md`).

---

## 4. System architecture

```
┌───────────────────────────────────────────────────────────────────────────┐
│ MAIN PROCESS (Node)                                                        │
│  App lifecycle · menus · dialogs · single-instance · file associations    │
│  ProjectService (open/save/autosave/migrate)                               │
│  MediaService (probe, import) ── JobQueue ── ThumbnailJob/WaveformJob/     │
│                                              ProxyJob/ReverseJob/...       │
│  ExportManager ── spawns hidden Export Window, owns encoder lifecycle      │
│  SettingsService · CacheService · FontService · Logger · HwAccelProbe      │
│  media:// protocol handler (streams local files with Range support)        │
└───────────────▲───────────────────────────────▲───────────────────────────┘
                │ typed IPC (preload bridge)    │ IPC
┌───────────────┴───────────────┐   ┌───────────┴────────────────────────────┐
│ EDITOR RENDERER (React)        │   │ EXPORT WINDOW (hidden, offscreen)       │
│  Stores: project/history/ui/   │   │  Node access enabled (trusted local     │
│          playback/jobs         │   │  code only, no remote content)          │
│  Panels: MediaBin, Preview,    │   │  DecoderPool: FFmpeg → raw RGBA frames  │
│          Timeline, Inspector…  │   │  Compositor (same code as preview)      │
│  PlaybackEngine: <video> pool, │   │  readPixels → encoder stdin             │
│          Web Audio graph       │   │                                         │
│  Compositor (WebGL2)           │   │                                         │
└───────────────┬────────────────┘   └───────────────┬─────────────────────────┘
                │ imports                            │ imports
        ┌───────┴────────────────────────────────────┴───────┐
        │ SHARED (pure TypeScript, no Electron/DOM deps*)      │
        │ types · time math · timeline operations · keyframes │
        │ evaluateFrame() · effect & transition registries     │
        │ audio graph builder · encoder args builder           │
        └──────────────────────────────────────────────────────┘
        * compositor/ uses WebGL and is shared by both renderers
```

### Key architectural principles

1. **One evaluation function.** `evaluateFrame(project, frame) → RenderGraph` (pure, in `shared/`) decides what is visible at a frame and with which resolved properties. Preview and export both call it.
2. **One compositor.** `Compositor.render(renderGraph, frameSources)` draws a frame with WebGL2. Preview feeds it `<video>` elements and images; export feeds it raw frames decoded by FFmpeg. Effects and transitions are implemented once, as GLSL shaders.
3. **Pure timeline operations.** Split, trim, move, ripple, etc. are pure functions `(project, args) → project` in `shared/timeline/`, unit-tested independently of the UI.
4. **Main process owns the file system and child processes.** The editor renderer never spawns processes; it calls typed IPC.
5. **Long work is a job.** Thumbnails, waveforms, proxies, reverse renders, silence detection, captions and export run as cancellable jobs with progress events.

### Security
- Editor window: `contextIsolation: true`, `nodeIntegration: false`, `sandbox: true`, strict CSP, no remote URLs.
- Export window: hidden, loads only the bundled local `export.html`, `nodeIntegration: true` is acceptable because it never loads remote content; it must not be navigable.
- IPC: every channel whitelisted in preload; every payload validated with Zod in main.
- Local media served via a custom `media://` protocol (registered privileged with `stream: true`), not `file://`.

---

## 5. Repository structure

```
clipforge/
├─ CLAUDE.md
├─ TECHNICAL_SPEC.md
├─ PROGRESS.md
├─ docs/
│  └─ DECISIONS.md
├─ package.json
├─ electron.vite.config.ts
├─ electron-builder.yml
├─ tsconfig.json / tsconfig.node.json / tsconfig.web.json
├─ vitest.config.ts · playwright.config.ts · .eslintrc.cjs · .prettierrc
├─ build/
│  └─ icon.ico
├─ resources/
│  └─ bin/                     # ffmpeg.exe, ffprobe.exe (if not using *-static)
├─ scripts/
│  ├─ fetch-ffmpeg.ts
│  └─ make-fixtures.ts         # generates test media with ffmpeg lavfi
├─ src/
│  ├─ main/
│  │  ├─ index.ts              # app lifecycle, single instance, file association
│  │  ├─ windows/
│  │  │  ├─ editorWindow.ts
│  │  │  └─ exportWindow.ts
│  │  ├─ menu.ts
│  │  ├─ protocol.ts           # media:// handler with HTTP Range support
│  │  ├─ ipc/
│  │  │  ├─ register.ts
│  │  │  ├─ project.ipc.ts  media.ipc.ts  jobs.ipc.ts  export.ipc.ts
│  │  │  └─ system.ipc.ts       # dialogs, fonts, settings, app info, shell
│  │  └─ services/
│  │     ├─ ffmpeg/  paths.ts  run.ts  probe.ts  progress.ts  hwaccel.ts
│  │     ├─ media/   importer.ts  thumbnails.ts  waveform.ts  proxy.ts  reverse.ts
│  │     ├─ project/ projectIO.ts  autosave.ts  recent.ts  migrations/
│  │     ├─ export/  exportManager.ts  audioRender.ts
│  │     ├─ jobs/    jobQueue.ts  jobTypes.ts
│  │     ├─ ai/      whisper.ts  silence.ts  scenes.ts           # later phases
│  │     ├─ cache.ts  settings.ts  fonts.ts  logger.ts
│  ├─ preload/
│  │  ├─ editor.ts             # contextBridge.exposeInMainWorld('api', …)
│  │  └─ api.d.ts
│  ├─ shared/
│  │  ├─ types/  project.ts  media.ts  ipc.ts  export.ts  jobs.ts  settings.ts
│  │  ├─ schema/ project.schema.ts  ipc.schema.ts
│  │  ├─ time.ts               # frames ↔ µs ↔ timecode, rational fps
│  │  ├─ ids.ts
│  │  ├─ timeline/
│  │  │  ├─ queries.ts         # clipsAt, clipEnd, trackOf, gaps…
│  │  │  ├─ operations.ts      # insert, overwrite, move, trim, split, ripple…
│  │  │  └─ snapping.ts
│  │  ├─ keyframes.ts          # interpolation & easing
│  │  ├─ evaluate/
│  │  │  ├─ evaluateFrame.ts   # project + frame → RenderGraph
│  │  │  └─ evaluateAudio.ts   # project → AudioPlan
│  │  ├─ effects/registry.ts   # effect definitions & parameter specs
│  │  ├─ transitions/registry.ts
│  │  ├─ export/
│  │  │  ├─ presets.ts
│  │  │  ├─ encoderArgs.ts     # ExportSettings → ffmpeg encoder args
│  │  │  └─ audioGraph.ts      # AudioPlan → ffmpeg filter_complex args
│  │  └─ constants.ts
│  ├─ compositor/              # WebGL2 — shared by editor + export window
│  │  ├─ Compositor.ts
│  │  ├─ FrameSource.ts        # interface + VideoElementSource, ImageSource, RawFrameSource
│  │  ├─ TexturePool.ts  FramebufferPool.ts
│  │  ├─ passes/ layerPass.ts  blendPass.ts  transitionPass.ts
│  │  ├─ shaders/ common.glsl.ts  layer.glsl.ts  blend.glsl.ts
│  │  ├─ effects/  colorCorrect.ts  blur.ts  chromaKey.ts  lut.ts  vignette.ts …
│  │  ├─ transitions/ dissolve.ts  wipe.ts  slide.ts  zoom.ts …
│  │  └─ text/TextRenderer.ts  # Canvas2D → texture
│  ├─ renderer/                # editor UI
│  │  ├─ index.html  main.tsx  App.tsx
│  │  ├─ store/  projectStore.ts  history.ts  uiStore.ts  playbackStore.ts
│  │  │          jobsStore.ts  selectionStore.ts
│  │  ├─ playback/ PlaybackEngine.ts  VideoElementPool.ts  AudioEngine.ts
│  │  ├─ timeline/ TimelineView.tsx  TimelineCanvas.ts  draw/*.ts
│  │  │            hitTest.ts  interactions/*.ts  viewport.ts
│  │  ├─ components/
│  │  │  ├─ layout/  AppShell.tsx  Splitters.tsx  MenuBar.tsx  StartScreen.tsx
│  │  │  ├─ mediaBin/ preview/ inspector/ effectsPanel/ transitionsPanel/
│  │  │  ├─ textPanel/ markers/ history/ captions/ transcript/
│  │  │  ├─ dialogs/  NewProject  Export  Settings  Shortcuts  Relink  About
│  │  │  └─ common/  NumberScrubber  ColorPicker  Timecode  Toasts …
│  │  ├─ shortcuts/ keymap.ts  useShortcuts.ts
│  │  └─ styles/
│  └─ exportWindow/
│     ├─ export.html
│     ├─ main.ts               # receives job, runs render loop
│     └─ DecoderPool.ts        # one FFmpeg decoder per active clip
└─ tests/
   ├─ unit/      (mirrors src/shared)
   ├─ golden/    (reference frames)
   └─ e2e/
```

---

## 6. Time model

Floating-point seconds cause drift and off-by-one cuts. Rules:

- **Timeline positions and durations** are integers in **project frames** (`Frame = number`, always an integer).
- **Project frame rate** is rational: `{ num: 30000, den: 1001 }` for 29.97, `{ num: 30, den: 1 }` for 30.
- **Source positions** (where in the media file a clip starts) are integers in **microseconds** (`sourceInUs`), because source fps may differ from project fps.
- Conversions live only in `shared/time.ts`:

```ts
framesToUs(frames, fps)        // Math.round(frames * 1e6 * fps.den / fps.num)
usToFrames(us, fps, mode)      // mode: 'floor' | 'round' | 'ceil'
framesToTimecode(frames, fps)  // "HH:MM:SS:FF" (non-drop-frame for v1)
timecodeToFrames(tc, fps)
sourceTimeAt(clip, frame, fps) // µs into source for a timeline frame, honoring speed & reverse
```

- A clip occupies frames `[start, start + duration)`.
- Source consumed by a clip = `framesToUs(duration) * speed`.
- Keyframe times are frames **relative to clip start**.
- 100% unit-test coverage for `time.ts`.

---

## 7. Data model

All types in `shared/types/project.ts` with matching Zod schemas.

```ts
export type ID = string;
export type Frame = number;            // integer
export type Rational = { num: number; den: number };

export interface Project {
  schemaVersion: number;               // starts at 1
  id: ID;
  name: string;
  createdAt: string;                   // ISO
  updatedAt: string;
  settings: ProjectSettings;
  media: Record<ID, MediaItem>;
  bins: Bin[];                         // folders in media panel
  tracks: Track[];                     // ordered, see note below
  clips: Record<ID, Clip>;
  transitions: Record<ID, Transition>;
  markers: Marker[];
  inPoint?: Frame;
  outPoint?: Frame;
}

export interface ProjectSettings {
  width: number;                       // even integer
  height: number;                      // even integer
  fps: Rational;
  sampleRate: 48000;
  backgroundColor: string;             // '#000000'
}

export interface Bin { id: ID; name: string; parentId?: ID }

export interface MediaItem {
  id: ID;
  name: string;
  path: string;                        // absolute
  relativePath?: string;               // relative to project file, for relinking
  kind: 'video' | 'audio' | 'image';
  fileSize: number;
  mtimeMs: number;
  cacheKey: string;                    // hash(path + size + mtime)
  durationUs: number;                  // 0 for images
  video?: {
    width: number; height: number;     // display size (rotation applied)
    fps: Rational; codec: string; pixFmt: string;
    rotation: 0 | 90 | 180 | 270; hasAlpha: boolean;
  };
  audio?: { codec: string; channels: number; sampleRate: number };
  previewable: boolean;                // Chromium can decode original directly
  proxy?: AssetStatus;                 // 720p H.264 preview file
  reversed?: AssetStatus;              // reversed render for reverse playback
  thumbnails?: { spritePath: string; tileW: number; tileH: number;
                 cols: number; count: number; intervalUs: number };
  waveform?: { peaksPath: string; samplesPerPeak: number; peakCount: number };
  binId?: ID;
  missing?: boolean;                   // runtime only, not saved
}

export interface AssetStatus {
  status: 'none' | 'queued' | 'running' | 'ready' | 'failed';
  path?: string; progress?: number; error?: string;
}

export interface Track {
  id: ID;
  kind: 'video' | 'audio' | 'caption';
  name: string;                        // "V1", "A1"…
  muted: boolean;  solo: boolean;  locked: boolean;  hidden: boolean;
  volumeDb: number;                    // audio tracks
  height: number;                      // px, UI only but persisted
}
// Video track order: tracks.filter(video) index 0 = V1 = bottom layer.
// Higher video tracks render on top.

export type Clip = MediaClip | TextClip | SolidClip | CaptionClip;

interface ClipBase {
  id: ID;
  trackId: ID;
  start: Frame;
  duration: Frame;                     // >= 1
  name?: string;
  color?: string;                      // label color
  enabled: boolean;
  linkGroupId?: ID;                    // linked A/V clips share this
}

export interface MediaClip extends ClipBase {
  type: 'media';
  mediaId: ID;
  sourceInUs: number;
  speed: number;                       // 0.1 – 10, default 1
  reversed: boolean;
  preservePitch: boolean;
  video?: VideoProps;                  // present on video tracks
  audio?: AudioProps;                  // present on audio tracks
  effects: EffectInstance[];
  keyframes: KeyframeMap;
}

export interface TextClip extends ClipBase {
  type: 'text';
  text: TextProps;
  video: VideoProps;
  effects: EffectInstance[];
  keyframes: KeyframeMap;
}

export interface SolidClip extends ClipBase {   // color matte / background
  type: 'solid';
  color: string;
  video: VideoProps;
  effects: EffectInstance[];
  keyframes: KeyframeMap;
}

export interface CaptionClip extends ClipBase { // lives on a 'caption' track
  type: 'caption';
  text: string;
  words?: { text: string; startUs: number; endUs: number }[];
}

export interface VideoProps {
  transform: {
    x: number; y: number;              // px offset from canvas center
    scale: number;                     // 1 = 100%
    scaleX?: number; scaleY?: number;  // if non-uniform
    rotation: number;                  // degrees
    anchorX: number; anchorY: number;  // 0–1 within the clip, default 0.5
  };
  opacity: number;                     // 0–1
  crop: { left: number; right: number; top: number; bottom: number }; // 0–1 fractions
  flipH: boolean; flipV: boolean;
  fit: 'fit' | 'fill' | 'stretch' | 'none';
  blendMode: 'normal' | 'multiply' | 'screen' | 'overlay' | 'add';
  fadeIn: Frame; fadeOut: Frame;       // opacity fades
}

export interface AudioProps {
  volumeDb: number;                    // -60 … +12, -Infinity = silent
  pan: number;                         // -1 … 1
  fadeIn: Frame; fadeOut: Frame;
}

export interface TextProps {
  content: string;                     // supports \n
  fontFamily: string; fontSize: number; fontWeight: number; italic: boolean;
  color: string; align: 'left' | 'center' | 'right';
  lineHeight: number; letterSpacing: number;
  stroke?: { color: string; width: number };
  shadow?: { color: string; blur: number; offsetX: number; offsetY: number };
  background?: { color: string; padding: number; radius: number };
  maxWidth?: number;                   // wrap width in px
  animationIn?: TextAnimation; animationOut?: TextAnimation;
}
export interface TextAnimation { preset: 'fade' | 'slideUp' | 'slideLeft' | 'pop' | 'typewriter'; duration: Frame }

export interface EffectInstance {
  id: ID;
  effectType: string;                  // key in effects registry
  enabled: boolean;
  params: Record<string, number | string | boolean>;
}

export type KeyframeMap = Partial<Record<AnimatablePath, Keyframe[]>>;
// AnimatablePath examples:
// 'video.transform.x' | 'video.transform.scale' | 'video.opacity' | 'audio.volumeDb'
// 'effect.<effectId>.<paramName>'
export type AnimatablePath = string;

export interface Keyframe {
  frame: Frame;                        // relative to clip start
  value: number;
  easing: 'linear' | 'easeIn' | 'easeOut' | 'easeInOut' | 'hold';
}

export interface Transition {
  id: ID;
  type: string;                        // key in transitions registry
  trackId: ID;
  fromClipId: ID | null;               // null = transition in from nothing (e.g. fade from black)
  toClipId: ID | null;                 // null = transition out to nothing
  duration: Frame;
  alignment: 'center' | 'startAtCut' | 'endAtCut';
  params: Record<string, number | string | boolean>;
}

export interface Marker { id: ID; frame: Frame; name: string; color: string; note?: string }
```

### Invariants (enforced by `shared/timeline/operations.ts`, checked by `validateProject()` in dev builds)
- Clips on the same track never overlap.
- A clip's source range stays within media duration (images and solids/text are unlimited).
- Linked clips share `start` and `duration` unless unlinked.
- A transition between two clips requires them to be adjacent on the same track; its duration is clamped to available handles (unused source media before/after the cut). If no handles exist, it is clamped and the user is notified.
- Track kinds match clip kinds (media clips with video props on video tracks, etc.).

### RenderGraph (output of `evaluateFrame`)

```ts
export interface RenderGraph {
  frame: Frame;
  width: number; height: number;
  background: string;
  layers: RenderLayer[];               // bottom → top
}

export type RenderLayer =
  | { kind: 'media'; clipId: ID; mediaId: ID; sourceTimeUs: number; props: ResolvedVideoProps; effects: ResolvedEffect[] }
  | { kind: 'image'; clipId: ID; mediaId: ID; props: ResolvedVideoProps; effects: ResolvedEffect[] }
  | { kind: 'text';  clipId: ID; text: TextProps; localFrame: Frame; props: ResolvedVideoProps; effects: ResolvedEffect[] }
  | { kind: 'solid'; clipId: ID; color: string; props: ResolvedVideoProps; effects: ResolvedEffect[] }
  | { kind: 'transition'; transitionType: string; progress: number; params: Record<string, unknown>;
      from: RenderLayer | null; to: RenderLayer | null };
```

`ResolvedVideoProps` = `VideoProps` with keyframes and fades applied for that frame. Captions render as text layers above all video tracks.

---

## 8. Project file format

- Extension: `.cfproj`. Content: UTF-8 JSON of `Project` (runtime-only fields stripped).
- Save is **atomic**: write `name.cfproj.tmp`, fsync, rename over the original; keep one `name.cfproj.bak`.
- On load: parse → run migrations (`migrations/1-to-2.ts`, …) up to the current `schemaVersion` → validate with Zod → check media paths exist (try `relativePath` if absolute path is missing) → mark `missing`.
- Autosave: every N minutes (default 2) to `%APPDATA%/ClipForge/autosave/<projectId>.cfproj` only if dirty. On startup, if an autosave is newer than the project file, offer recovery.
- Unknown fields are preserved (forward compatibility).

---

## 9. State management and undo/redo

### Stores (Zustand)
| Store | Contents | In undo history? |
|---|---|---|
| `projectStore` | `Project`, file path, dirty flag | **Yes** |
| `selectionStore` | selected clip/transition/keyframe IDs | No |
| `uiStore` | panel sizes, timeline zoom & scroll, active tool, snapping on/off, theme | No |
| `playbackStore` | playhead frame, isPlaying, shuttle speed, preview quality | No |
| `jobsStore` | background jobs & progress | No |

### Undo/redo design
- Every project mutation goes through `projectStore.apply(label, recipe)` which uses Immer `produceWithPatches`, pushing `{ label, patches, inversePatches }` onto the history stack.
- **Transactions** for continuous gestures (dragging, trimming, scrubbing a slider): `beginTransaction(label)` → many `apply` calls (merged) → `commitTransaction()` produces **one** undo entry; `cancelTransaction()` reverts.
- History limit: 500 entries (configurable).
- Redo stack clears on a new action.
- Undo/redo also restores a sensible selection (stored alongside the entry).
- Labels appear in Edit menu ("Undo Split Clip") and in the History panel.

All timeline edits are implemented as pure functions in `shared/timeline/operations.ts`, e.g.:

```ts
splitClip(p, clipId, atFrame): Project
trimClipStart(p, clipId, newStart, opts: { ripple: boolean }): Project
trimClipEnd(p, clipId, newEnd, opts): Project
moveClips(p, clipIds, deltaFrames, targetTrackId?): Project      // overwrite semantics
rippleDelete(p, clipIds): Project
insertMedia(p, mediaId, trackId, atFrame, mode: 'insert' | 'overwrite'): Project
rollEdit(p, leftClipId, rightClipId, delta): Project
slipClip(p, clipId, deltaUs): Project
slideClip(p, clipId, delta): Project
closeGaps(p, trackId): Project
setSpeed(p, clipId, speed, opts: { ripple: boolean }): Project
addTransition(p, trackId, atCutFrame, type, duration): Project
link(p, clipIds) / unlink(p, clipIds)
```

Each has unit tests including edge cases (frame 0, adjacent clips, linked clips, locked tracks, transitions attached to affected clips).

---

## 10. IPC contract

Defined once in `shared/types/ipc.ts`; preload exposes `window.api` with exactly these methods. Payloads validated with Zod in main.

```ts
interface ClipForgeAPI {
  app: {
    info(): Promise<{ version: string; ffmpegVersion: string; hwEncoders: HwEncoder[] }>;
    openLogsFolder(): Promise<void>;
    onOpenFile(cb: (path: string) => void): Unsubscribe;   // file association / second instance
  };
  project: {
    open(path?: string): Promise<{ path: string; project: Project; missing: ID[] } | null>;
    save(path: string, project: Project): Promise<void>;
    saveAs(project: Project): Promise<string | null>;
    autosave(project: Project): Promise<void>;
    checkRecovery(): Promise<RecoveryCandidate[]>;
    recent(): Promise<RecentProject[]>;
  };
  media: {
    pickFiles(): Promise<string[]>;
    import(paths: string[]): Promise<MediaItem[]>;        // probe + queue thumb/waveform/proxy jobs
    relink(mediaId: ID, newPath: string): Promise<MediaItem>;
    requestProxy(mediaId: ID): Promise<void>;
    requestReversed(mediaId: ID): Promise<void>;
    readPeaks(peaksPath: string): Promise<ArrayBuffer>;
    toMediaUrl(path: string): string;                      // media://local/<encoded path>
  };
  jobs: {
    list(): Promise<JobInfo[]>;
    cancel(jobId: ID): Promise<void>;
    onUpdate(cb: (job: JobInfo) => void): Unsubscribe;
  };
  export: {
    start(project: Project, settings: ExportSettings): Promise<ID>;   // jobId
    cancel(jobId: ID): Promise<void>;
    onProgress(cb: (p: ExportProgress) => void): Unsubscribe;
    snapshotFrame(project: Project, frame: Frame, outPath: string): Promise<void>;
  };
  system: {
    fonts(): Promise<string[]>;
    settings(): Promise<Settings>;
    setSettings(patch: Partial<Settings>): Promise<Settings>;
    showSaveDialog(opts): Promise<string | null>;
    showOpenDialog(opts): Promise<string[]>;
    revealInExplorer(path: string): Promise<void>;
    openPath(path: string): Promise<void>;
    clearCache(): Promise<void>;
  };
  ai: {                                                    // phase 5+
    transcribe(mediaId: ID, opts: { language?: string; model: 'base' | 'small' }): Promise<ID>;
    detectSilence(mediaId: ID, opts: { thresholdDb: number; minSilenceMs: number }): Promise<{ startUs: number; endUs: number }[]>;
    detectScenes(mediaId: ID, opts: { threshold: number }): Promise<number[]>;
  };
}
```

Main → renderer events use `webContents.send` on whitelisted channels: `jobs:update`, `export:progress`, `app:open-file`, `menu:command` (menu clicks dispatch commands by ID, e.g. `edit.undo`).

---

## 11. Media pipeline

### 11.1 Import flow
1. User drops/picks files → `media.import(paths)`.
2. For each path: `ffprobe -v error -print_format json -show_format -show_streams <path>`.
3. Classify `kind`; read duration, dimensions (apply `rotate` side data), fps (`avg_frame_rate`, fall back to `r_frame_rate`), codecs, channels, alpha (pix_fmt contains `a`).
4. Decide `previewable` (Chromium can decode it):
   - Video: container mp4/mov/webm/m4v **and** codec in {h264, vp8, vp9, av1} (hevc only if a runtime `MediaSource.isTypeSupported`/`canPlayType` probe in the renderer says yes), pixel format 8-bit 4:2:0. Anything else (MKV, AVI, ProRes, DNxHD, 10-bit, MPEG-2, variable-frame-rate screen recordings flagged by large frame-rate mismatch) → **not previewable → proxy required**.
   - Audio: mp3, aac, m4a, wav, flac, ogg/opus are previewable.
   - Images: png, jpg, webp previewable; GIF → converted to a video proxy.
5. Create `MediaItem`, add to project (undoable "Import Media").
6. Queue jobs: thumbnails (video/image), waveform (anything with audio), proxy (if not previewable, or if > 1080p and "auto proxy" setting is on).
7. Unsupported/corrupt files: toast with reason, item not added.

### 11.2 Job queue (`main/services/jobs/jobQueue.ts`)
- Priority queue: export > thumbnails > waveform > proxy > reversed > AI.
- Concurrency: 2 by default (configurable); export runs alone at full priority.
- Each job: `{ id, type, mediaId?, status, progress (0–1), etaSec?, error? }`; emits `jobs:update`.
- Cancellable: kills its child process and deletes partial outputs.
- Results written to cache; on completion the main process returns the updated `AssetStatus`, and the renderer updates the media item **without** creating an undo entry.

### 11.3 Cache
- Root: `%APPDATA%/ClipForge/cache/<cacheKey>/` containing `thumbs.jpg`, `thumbs.json`, `peaks.bin`, `proxy.mp4`, `reversed.mp4`.
- Reused across projects for the same file (same path+size+mtime).
- Settings: change cache location, show size, clear cache.

### 11.4 Thumbnails
- Sprite sheet: `ffmpeg -i in -vf "fps=1/<interval>,scale=160:-2,tile=10x<rows>" -frames:v 1 -q:v 4 thumbs.jpg` (split into multiple sprites for long media). Interval chosen so there are ≤ 300 thumbnails.
- Bin thumbnail = frame at 10% duration.

### 11.5 Waveforms
- Decode mono 16-bit PCM at 8 kHz: `ffmpeg -i in -vn -ac 1 -ar 8000 -f s16le pipe:1`.
- Compute min/max peaks per 64 samples, stored as `Int16Array` pairs in `peaks.bin`; build lower-resolution levels on the fly in the renderer for zoomed-out views.

### 11.6 Proxies
- `ffmpeg -i in -vf "scale=-2:720:flags=bicubic,format=yuv420p" -c:v libx264 -preset veryfast -crf 23 -g 15 -c:a aac -b:a 128k -movflags +faststart proxy.mp4`
- Short GOP (`-g 15`) makes seeking responsive.
- Constant frame rate output (`-fps_mode cfr`, `-r <source fps>`) to fix variable-frame-rate phone/screen recordings.
- Use hardware encoder for proxies when available (Section 14.6).
- Preview uses the proxy when `settings.useProxies` is on **or** the original is not previewable. Export **always** uses the original.

### 11.7 Reverse playback asset
- HTML video cannot play backwards smoothly. When a clip is set to reversed, queue `reversed.mp4` for the clip's source range (`-vf reverse -af areverse`, segmented for long sources). Preview shows a "preparing" overlay until ready. Export decodes the original in reverse via the same reversed asset or segment-wise decode.

---

## 12. Compositor (shared rendering core)

### 12.1 Responsibilities
`Compositor` takes a `RenderGraph` and a `FrameSourceProvider` and draws the frame into a canvas (preview) or framebuffer (export).

```ts
interface FrameSource {               // anything that can become a texture
  kind: 'video-element' | 'image' | 'raw-rgba' | 'canvas';
  width: number; height: number;
  upload(gl: WebGL2RenderingContext, tex: WebGLTexture): void;
}
interface FrameSourceProvider {
  get(layer: RenderLayer): FrameSource | null;   // null = not ready → draw placeholder (preview) / wait (export)
}
class Compositor {
  constructor(canvas: HTMLCanvasElement | OffscreenCanvas, opts: { width; height; quality: 1 | 0.5 | 0.25 });
  render(graph: RenderGraph, sources: FrameSourceProvider): void;
  readPixels(out: Uint8Array): void;            // export only (RGBA, top-down)
  resize(w, h, quality): void;
  dispose(): void;
}
```

### 12.2 Render algorithm (per frame)
1. Clear the output framebuffer to `background`.
2. For each layer bottom → top:
   - Get texture from source (or `TextRenderer` for text, solid color for solids).
   - **Layer pass:** draw into a project-sized temp framebuffer: apply crop → fit mode → flip → transform matrix (anchor, scale, rotation, translate) → output premultiplied RGBA.
   - **Effects passes:** ping-pong framebuffers for each enabled effect in order.
   - **Transition layers:** render `from` and `to` layers into two framebuffers, then run the transition shader with `progress`.
   - **Blend pass:** composite onto the accumulator with the layer's opacity and blend mode.
3. Present (preview) or `readPixels` (export).

All math in **linear-ish sRGB with premultiplied alpha**; colors must be consistent between preview and export because they use the same code path.

### 12.3 Effects registry
`shared/effects/registry.ts` defines metadata; `compositor/effects/*.ts` implements shaders.

```ts
interface EffectDefinition {
  type: string;                        // 'colorCorrect'
  name: string; category: 'Color' | 'Blur & Sharpen' | 'Stylize' | 'Keying' | 'Distort';
  params: ParamSpec[];                 // { name, label, kind: 'number'|'color'|'bool'|'enum'|'file', min, max, step, default, animatable }
  phase: number;                       // which build phase introduces it
}
```

Initial effects: colorCorrect (exposure, brightness, contrast, saturation, temperature, tint, highlights, shadows), gaussianBlur (separable, 2 passes), sharpen, vignette, blackWhite, sepia, invert, filmGrain (seeded by frame for determinism), lut3d (`.cube` parsed in main, uploaded as 3D texture), chromaKey (key color, tolerance, softness, spill), colorWheels, curves, mask (rect/ellipse, feather, invert).

### 12.4 Transitions registry
Each transition is a shader `(from, to, progress, params) → color`. Initial set: crossDissolve, dipToBlack, dipToWhite, wipe (direction, softness), slide (direction), push, zoom, blurDissolve. Audio under a transition gets an equal-power crossfade (handled in audio plan).

### 12.5 Text rendering
- `TextRenderer` draws `TextProps` into an `OffscreenCanvas` 2D context (fonts are system fonts loaded via CSS `local()`), measures and wraps lines, draws background box, shadow, stroke, fill; then uploads as a texture.
- Cache rendered bitmaps by hash of `TextProps` + animation state; redraw only when changed.
- Text animations (typewriter, etc.) are computed from `localFrame`, so preview and export match.

### 12.6 Determinism
- Given the same project and frame, the compositor must produce identical output in preview (full quality) and export. Random effects (grain) are seeded by `(clipId, frame)`.
- Golden-frame tests enforce this (Section 19.3).

---

## 13. Preview playback engine

### 13.1 Components
- `PlaybackEngine` — owns the clock, schedules rendering, coordinates video and audio.
- `VideoElementPool` — `<video>` elements (muted) keyed by clip ID, sourced from `media://` URLs (proxy or original). Max ~16 live elements; LRU eviction.
- `AudioEngine` — Web Audio graph: per audio clip an `<audio>` element → `MediaElementAudioSourceNode` → clip `GainNode` (volume + fades) → `StereoPannerNode` → track `GainNode` → master `GainNode` → `AnalyserNode` (meters) → destination.
- Images decoded once into `ImageBitmap`, cached.

### 13.2 Clock
- While playing, the master clock is `AudioContext.currentTime` (anchored at play start): `frame = startFrame + floor((ctx.currentTime - anchor) * fps * shuttleSpeed)`.
- Each `requestAnimationFrame`:
  1. Compute `frame`; update `playbackStore.playhead` (throttled for React).
  2. `graph = evaluateFrame(project, frame)`.
  3. For each media layer, ensure its pooled video element is playing at `playbackRate = clip.speed * shuttleSpeed` and is within tolerance of `sourceTimeUs` (drift > 2 frames → hard `currentTime` seek).
  4. Ensure audio elements for clips active now are playing and in sync (drift > 40 ms → resync); fade/volume via `GainNode.gain.setValueAtTime` / ramps scheduled from the audio plan.
  5. **Pre-roll:** clips starting within the next 2 seconds get their elements created, seeked to their in-point and paused, ready to start.
  6. `compositor.render(graph, provider)`.
- If rendering falls behind, **drop frames, never desync audio**.

### 13.3 Paused / scrubbing
- On playhead change while paused: seek each needed video element to the exact source time; wait for `requestVideoFrameCallback` (or `seeked`) on all; then render. Coalesce rapid scrubs — only the latest requested frame renders.
- Show the last rendered frame while seeking (no flashing).
- Optional audio scrubbing: play a 60 ms snippet at the playhead when scrubbing (setting).

### 13.4 Shuttle (J/K/L)
- L: 1×, 2×, 4×, 8× forward. J: reverse at 1×, 2×, 4×, 8× — implemented by stepping paused frames on a timer (render-only, audio muted), since media elements cannot play backwards.
- K: pause. K+J / K+L: step one frame.

### 13.5 Preview quality
Full / Half / Quarter scale the compositor's internal resolution; also used automatically when frames are dropped repeatedly (adaptive mode, can be disabled).

### 13.6 Limits
Chromium allows a limited number of simultaneous media players; never create more than the pool size. Release elements for clips far from the playhead.

---

## 14. Export engine

### 14.1 Overview
Export is **frame-by-frame rendering through the same compositor**, encoded by FFmpeg. Audio is mixed separately by FFmpeg from an audio plan and muxed in.

```
ExportManager (main)
  1. Validate project (missing media, empty range, odd dimensions)
  2. Render audio  ── buildAudioGraph(plan) ──► ffmpeg ──► temp/audio.wav
  3. Spawn hidden Export Window, send {project, settings, range, audioPath}
Export Window
  4. Start encoder:  ffmpeg -f rawvideo -pix_fmt rgba -s WxH -r FPS -i pipe:0
                            -i audio.wav  <encoder args>  out.tmp.mp4
  5. For frame = in … out-1:
        graph = evaluateFrame(project, frame)
        sources = DecoderPool.framesFor(graph)        // exact frames, awaited
        compositor.render(graph, sources)
        compositor.readPixels(buffer)
        await writeWithBackpressure(encoder.stdin, buffer)
        post progress every 10 frames
  6. Close stdin, wait for encoder exit, rename out.tmp.mp4 → final name
  7. Report success / failure; clean temp files
```

### 14.2 DecoderPool (frame-accurate decoding)
- One FFmpeg decoder process per media clip that is active during export; created just before the clip is needed (pre-start 1 second early), killed after the clip ends.
- Decoder command (example, forward playback):
  ```
  ffmpeg -ss <sourceInSec> -i <original path> -an
         -vf "setpts=(PTS-STARTPTS)/<speed>,fps=<project fps>,scale=<w>:<h>:flags=bicubic:in_color_matrix=auto,format=rgba"
         -f rawvideo pipe:1
  ```
  - `<w>:<h>` = media display size, capped so neither dimension exceeds 2× the project size (saves bandwidth; GPU does final scaling).
  - Rotation side data is honored automatically by FFmpeg (autorotate).
  - Reversed clips decode from the reversed asset or a `reverse` filter on bounded segments.
- Reader pulls exactly `w*h*4` bytes per frame from stdout into a reusable `Buffer`; `RawFrameSource.upload` calls `texSubImage2D`.
- Images: loaded once as `ImageBitmap`.
- If a decoder ends early (e.g., source shorter than expected), hold the last frame and log a warning.

### 14.3 Audio plan and graph
- `evaluateAudio(project, range) → AudioPlan` lists audio segments: source path, `sourceInUs`, `durationUs`, timeline offset, speed, reversed, volume curve (static dB, fades, keyframes, transition crossfades, track gain, mute/solo, master), pan.
- `buildAudioGraph(plan) → string[]` (pure, unit-tested) produces FFmpeg args, per segment:
  `atrim` → `asetpts=PTS-STARTPTS` → `atempo` chain for speed (with pitch preservation; `asetrate`+`aresample` when pitch not preserved) → `areverse` if reversed → `volume=…:eval=frame` (expression for keyframes/fades) → `pan` → `adelay=<offset ms>|<offset ms>` → then `amix=inputs=N:normalize=0:dropout_transition=0` → master `volume` → `aresample=48000` → `apad` / `atrim` to exact export length.
- Output `pcm_s16le` 48 kHz stereo WAV in a temp dir. Tested by comparing output duration and RMS at known points against fixtures.
- Preview `AudioEngine` uses the same `AudioPlan` for gains and fades so levels match.

### 14.4 ExportSettings

```ts
interface ExportSettings {
  outputPath: string;
  container: 'mp4' | 'mov' | 'webm' | 'gif' | 'mp3' | 'wav' | 'png';   // png = current frame
  videoCodec: 'h264' | 'hevc' | 'vp9' | 'prores' | 'gif';
  audioCodec: 'aac' | 'opus' | 'mp3' | 'pcm';
  width: number; height: number;       // may differ from project (scaled; aspect preserved, padded)
  fps: Rational;
  quality: { mode: 'crf'; value: number } | { mode: 'bitrate'; kbps: number };
  audioBitrateKbps: number;            // 192 default
  range: 'all' | 'inOut';
  hardwareEncoding: 'auto' | 'off';
  burnInCaptions: boolean;
  exportSrt: boolean;
}
```

Presets (`shared/export/presets.ts`): YouTube 1080p (H.264, CRF 18, 30/60 fps match project), YouTube 4K, Instagram Reel / TikTok / YouTube Shorts (1080×1920), Instagram Square (1080×1080), Twitter/X (1280×720, ≤ 512 MB), Archive (ProRes 422 MOV), GIF (480 px wide, 15 fps, palettegen/paletteuse), Audio only (MP3 320 / WAV).

### 14.5 Encoder args (`buildEncoderArgs`, pure)
- H.264 software: `-c:v libx264 -preset medium -crf <n> -pix_fmt yuv420p -profile:v high -movflags +faststart`
- Always tag BT.709: `-vf scale=out_color_matrix=bt709:out_range=tv,format=yuv420p -colorspace bt709 -color_primaries bt709 -color_trc bt709`
- AAC: `-c:a aac -b:a 192k`
- Output to `*.tmp.<ext>`, rename on success.

### 14.6 Hardware encoding
- At startup (cached per FFmpeg version), detect working encoders by running a 1-frame test encode per candidate (`h264_nvenc`, `hevc_nvenc`, `h264_qsv`, `hevc_qsv`, `h264_amf`, `hevc_amf`) using `-f lavfi -i color=black:s=256x256:d=0.1 -f null -`.
- `auto` picks NVENC > QSV > AMF > libx264. Map CRF to the encoder's quality control (`-cq` for NVENC, `-global_quality` for QSV, `-qp_i/-qp_p` for AMF).
- If a hardware encode fails mid-export, retry once with software and tell the user.

### 14.7 Progress, cancel, failure
- Progress = frames written / total frames; ETA = rolling average of the last 120 frames. Shows phase ("Mixing audio", "Rendering video", "Finalizing").
- Cancel: kill encoder, decoders, close the export window; delete temp/partial files.
- On failure: show a readable message; full FFmpeg stderr goes to the log; "Copy details" button.
- On success: toast with "Play" and "Show in folder"; Windows notification if the app is not focused.
- Keep the editor fully usable during export (the export window has its own GPU context).

### 14.8 Export verification (automatic)
After encoding, run FFprobe on the output and verify duration (±1 frame of expected), resolution, and stream presence. Log a warning if not matched.

---

## 15. Feature requirements

Format: **ID — feature — phase**. Phases are defined in Section 20.

### 15.1 Project management
| ID | Feature | Phase |
|---|---|---|
| P-1 | Start screen: New Project, Open Project, recent projects list | 1 |
| P-2 | New project dialog: name, resolution presets (1920×1080, 1280×720, 3840×2160, 1080×1920, 1080×1080, custom), fps (23.976, 24, 25, 29.97, 30, 50, 59.94, 60), background color | 1 |
| P-3 | Save / Save As / Open `.cfproj` (atomic save, backup) | 1 |
| P-4 | Unsaved-changes indicator (`*` in title), prompt on close/open/new | 1 |
| P-5 | Autosave + crash recovery prompt | 2 |
| P-6 | Missing media detection and Relink dialog (with "search this folder for other missing files") | 2 |
| P-7 | Change project settings after creation (resolution, fps, background) | 3 |
| P-8 | "Collect project": copy project + all used media into one folder | 4 |
| P-9 | Open `.cfproj` by double-clicking in Explorer (file association), single instance | 1 |

### 15.2 Media import & bin
| ID | Feature | Phase |
|---|---|---|
| M-1 | Import via menu, button, `Ctrl+I`, drag-and-drop from Explorer (files and folders) | 1 |
| M-2 | Formats: MP4, MOV, MKV, WebM, AVI, M4V; MP3, WAV, AAC, M4A, FLAC, OGG, OPUS; PNG, JPG, WebP, GIF | 1 |
| M-3 | FFprobe metadata; info tooltip (resolution, fps, codec, duration, size) | 1 |
| M-4 | Bin thumbnail; status badge for jobs (proxy generating, failed, missing) | 1 |
| M-5 | Grid/list views, sort (name, date, duration, type), search, thumbnail size slider | 2 |
| M-6 | Remove media (warn if used; option to remove its clips too) | 1 |
| M-7 | Folders (bins) in the media panel; drag items between folders | 3 |
| M-8 | "Used on timeline" indicator; "Reveal in timeline" | 3 |
| M-9 | Hover-scrub preview on video thumbnails | 3 |
| M-10 | Automatic proxies for non-previewable media; optional proxies for > 1080p; "Use proxies" toggle | 1 (required) / 3 (optional) |
| M-11 | Source monitor: double-click a bin item to preview it in the player with its own In/Out, then Insert (`,`) / Overwrite (`.`) to timeline | 3 |
| M-12 | Built-in solid color and text generators in the bin's "New" menu | 2 |

### 15.3 Timeline
| ID | Feature | Phase |
|---|---|---|
| T-1 | Multi-track timeline: default V1, V2, A1, A2; add/delete/rename/reorder tracks | 1 |
| T-2 | Drag media from bin to timeline; video+audio files create linked clips on paired tracks; auto-create tracks if dropped below/above existing | 1 |
| T-3 | Move clips by drag, within and across same-kind tracks (overwrite semantics) | 1 |
| T-4 | Trim edges by drag (bounded by source length; cursor changes on edges) | 1 |
| T-5 | Split at playhead (`S` / `Ctrl+K`); split all tracks at playhead (`Shift+S`); Razor tool (`C`) | 1 |
| T-6 | Delete (leave gap) and Ripple Delete (`Shift+Delete`); delete a gap by selecting it | 1 |
| T-7 | Snapping to clip edges, playhead, markers, In/Out (toggle `N`); visual snap line | 1 |
| T-8 | Zoom (slider, `Ctrl+wheel`, `+`/`-`, `\` fit) and scroll (wheel, `Shift+wheel`, auto-scroll when dragging near edges and during playback) | 1 |
| T-9 | Ruler with timecode; click/drag ruler to move playhead | 1 |
| T-10 | Clip thumbnails (filmstrip) and audio waveforms | 2 |
| T-11 | Multi-select: click, `Ctrl`/`Shift` click, marquee; select all (`Ctrl+A`); select forward from playhead (`A`) | 2 |
| T-12 | Copy, cut, paste at playhead (to targeted track), duplicate | 2 |
| T-13 | Link/unlink A/V (`Ctrl+L`); `Alt`-drag edits one side of a linked pair | 2 |
| T-14 | Track headers: mute, solo, hide, lock, height resize, target track toggle | 2 |
| T-15 | Ripple trim (`Ctrl`-drag edge or Ripple tool `B`) and Roll trim (Roll tool `R`) | 3 |
| T-16 | Slip (`Y`) and Slide (`U`) tools with dual-frame preview | 4 |
| T-17 | Markers: add (`M`), rename, color, note, jump (`Shift+M` / `Ctrl+Shift+M`), markers list panel | 2 |
| T-18 | In/Out points on timeline (`I`/`O`, clear with `Alt+X`) | 3 |
| T-19 | Insert and overwrite edits from bin/source monitor at playhead | 3 |
| T-20 | Close gaps on track / all tracks | 3 |
| T-21 | Freeze frame (creates a still clip of the frame at playhead, default 2 s) | 3 |
| T-22 | Clip context menu: speed, reverse, detach audio, freeze frame, rename, label color, enable/disable, reveal in bin, properties | 2 |
| T-23 | Clip label colors; disabled clips appear dimmed and are skipped in preview/export | 2 |
| T-24 | Nested sequences / compound clips | 6 |

### 15.4 Preview
| ID | Feature | Phase |
|---|---|---|
| V-1 | Composited preview of the timeline at playhead | 1 |
| V-2 | Play/pause (`Space`), go to start/end (`Home`/`End`), previous/next edit (`↑`/`↓`) | 1 |
| V-3 | Frame step (`←`/`→`), 1-second/10-frame step (`Shift+←/→`) | 1 |
| V-4 | J/K/L shuttle | 2 |
| V-5 | Audio playback mixed from all audible tracks, in sync | 1 |
| V-6 | Current timecode (click to type a timecode and jump) and total duration | 1 |
| V-7 | Preview quality Full/Half/Quarter + adaptive | 3 |
| V-8 | Fullscreen preview (`F`), fit/100%/200% zoom of the canvas | 2 |
| V-9 | Overlays: title/action safe areas, center cross, rule of thirds | 3 |
| V-10 | Loop In/Out playback (`Ctrl+L` when player focused / button) | 3 |
| V-11 | Audio meters (per channel peak + hold) | 3 |
| V-12 | Drop frames, never desync audio | 1 |
| V-13 | On-canvas transform handles for the selected clip (move, scale, rotate) | 2 |
| V-14 | Export current frame as PNG | 3 |

### 15.5 Clip properties (Inspector)
| ID | Feature | Phase |
|---|---|---|
| C-1 | Transform: position, scale (uniform/non-uniform), rotation, anchor | 2 |
| C-2 | Opacity | 2 |
| C-3 | Crop | 2 |
| C-4 | Flip H/V | 2 |
| C-5 | Fit mode: Fit, Fill, Stretch, None (default: Fit) | 2 |
| C-6 | Speed 0.1×–10× with presets; ripple option; reverse; preserve pitch | 3 |
| C-7 | Volume (dB) | 1 |
| C-8 | Pan | 2 |
| C-9 | Audio fade handles on clip corners; video fade in/out | 2 |
| C-10 | Blend modes | 4 |
| C-11 | Reset per group; copy/paste attributes between clips (`Ctrl+Alt+V`) | 2 / 3 |
| C-12 | Number fields support drag-to-scrub, arrow keys, typed math (`+10`, `*2`) | 2 |

### 15.6 Keyframes
| ID | Feature | Phase |
|---|---|---|
| K-1 | Stopwatch toggle per animatable property | 3 |
| K-2 | Add/remove/jump to keyframes; diamonds on clip in timeline and in Inspector mini-timeline | 3 |
| K-3 | Interpolation: linear, ease in, ease out, ease in-out, hold | 3 |
| K-4 | Drag keyframes in time; multi-select keyframes | 4 |
| K-5 | Motion path shown on canvas for animated position | 4 |
| K-6 | Volume keyframes as rubber-band line on audio clips (`Ctrl+click` to add) | 4 |

### 15.7 Text & titles
| ID | Feature | Phase |
|---|---|---|
| X-1 | Add text clip (`T` or Text panel); edit inline on canvas or in Inspector | 2 |
| X-2 | Font family (system fonts), size, weight, italic, color, align, line height, letter spacing, wrap width | 2 |
| X-3 | Stroke, shadow, background box (padding, radius) | 2 |
| X-4 | Title presets: centered title, lower third, subscribe banner, end credits (scrolling) | 3 |
| X-5 | In/out animation presets: fade, slide, pop, typewriter | 4 |
| X-6 | Save custom text style as preset | 4 |

### 15.8 Transitions
| ID | Feature | Phase |
|---|---|---|
| R-1 | Transitions panel with animated previews on hover; drag onto cut or clip edge | 3 |
| R-2 | Cross dissolve, dip to black, dip to white, wipe, slide, push, zoom, blur dissolve | 3 |
| R-3 | Duration by dragging edges or typing; alignment (center / start / end at cut) | 3 |
| R-4 | Audio crossfade (equal power) on linked audio; standalone audio crossfades | 3 |
| R-5 | Apply default transition at nearest cut (`Ctrl+T`); apply to all selected cuts | 3 |
| R-6 | Handle warning when source media is too short | 3 |

### 15.9 Effects & color
| ID | Feature | Phase |
|---|---|---|
| E-1 | Effects panel with categories and search; drag onto clip; stack in Inspector (reorder, toggle, delete, reset) | 3 |
| E-2 | Color correction: exposure, brightness, contrast, saturation, temperature, tint, highlights, shadows | 3 |
| E-3 | Blur, sharpen, vignette, black & white, sepia, invert, film grain | 3 |
| E-4 | LUT (`.cube`) with intensity | 4 |
| E-5 | Chroma key with spill suppression; matte view toggle | 4 |
| E-6 | Color wheels (lift/gamma/gain), RGB curves | 5 |
| E-7 | Scopes: waveform, vectorscope, histogram (computed on GPU from preview frame) | 5 |
| E-8 | Stabilization (FFmpeg `vidstabdetect`/`vidstabtransform` as a job producing a stabilized intermediate) | 5 |
| E-9 | Masks (rectangle, ellipse) with feather, expansion, invert | 5 |
| E-10 | Adjustment layer clip (effects apply to everything below) | 4 |

### 15.10 Audio
| ID | Feature | Phase |
|---|---|---|
| A-1 | Clip volume | 1 |
| A-2 | Track volume, mute, solo | 2 |
| A-3 | Master volume + master meter | 3 |
| A-4 | Detach audio from video | 2 |
| A-5 | Normalize loudness (EBU R128 analysis job; target −14 LUFS default) | 4 |
| A-6 | Noise reduction (`afftdn`, applied via pre-rendered cleaned audio asset) | 4 |
| A-7 | EQ (3-band), compressor | 5 |
| A-8 | Auto-ducking music under voice | 5 |
| A-9 | Voiceover recording to new audio track with countdown | 4 |

### 15.11 Export
| ID | Feature | Phase |
|---|---|---|
| O-1 | Export dialog: name, folder, preset, format, resolution, fps, quality, estimated file size | 1 |
| O-2 | MP4 H.264/AAC | 1 |
| O-3 | More formats: H.265 MP4, WebM VP9/Opus, ProRes MOV, GIF, MP3/WAV, PNG frame | 3 |
| O-4 | Presets (Section 14.4) | 2 |
| O-5 | Progress with phase, %, elapsed, ETA, cancel | 1 |
| O-6 | Export In/Out range | 3 |
| O-7 | Hardware encoding auto-detect with fallback | 3 |
| O-8 | Completion toast + Windows notification; Play / Show in folder | 2 |
| O-9 | Export queue | 4 |
| O-10 | Export `.srt`, burn-in captions | 5 |
| O-11 | Output matches preview (golden-frame tests) | 1+ |

### 15.12 Editing infrastructure
| ID | Feature | Phase |
|---|---|---|
| U-1 | Undo/redo for all project changes, with transactions for gestures | 1 |
| U-2 | Keyboard shortcuts (Section 17) | 1 |
| U-3 | Shortcut editor (search, rebind, conflict warning, reset) | 3 |
| U-4 | Context menus (clips, tracks, bin items, transitions, markers) | 2 |
| U-5 | Settings dialog (Section 16.8) | 2 |
| U-6 | Toasts for jobs/errors; jobs popover listing background jobs with cancel | 1 |
| U-7 | Logs + Help → Open logs folder | 1 |
| U-8 | About dialog with app version, FFmpeg version and licence notice | 1 |
| U-9 | History panel (click an entry to jump back) | 4 |
| U-10 | Resizable panels with layout persistence; Window → Reset layout | 1 |
| U-11 | Tooltips with shortcut hints on all toolbar buttons | 2 |

### 15.13 AI-assisted (local only, phase 5+)
| ID | Feature | Phase |
|---|---|---|
| AI-1 | Auto-captions via `whisper.cpp` (Windows binary downloaded on first use to `%APPDATA%/ClipForge/models`, with consent dialog showing download size); captions land on a caption track as editable CaptionClips | 5 |
| AI-2 | Caption styles (uses text props); word-by-word highlight style | 5 |
| AI-3 | Silence removal: `silencedetect` → preview list of cuts (threshold, min silence, padding) → apply as one undoable ripple edit | 5 |
| AI-4 | Scene detection (`select='gt(scene,T)'`) → split clip at scene changes | 5 |
| AI-5 | Transcript panel; delete words to cut the timeline (text-based editing) | 6 |
| AI-6 | Background removal (local segmentation model via ONNX Runtime Web/WebGPU) | 6 |
| AI-7 | Auto-reframe to 9:16 following the subject | 6 |

---

## 16. UI specification

### 16.1 Main layout
```
┌──────────────────────────────────────────────────────────────────────────┐
│ Menu: File  Edit  Clip  Timeline  View  Window  Help        [jobs ⟳ 2]    │
├───────────────────┬────────────────────────────────┬─────────────────────┤
│ Tabs:             │  PREVIEW                        │ INSPECTOR           │
│ Media | Text |    │  ┌──────────────────────────┐   │ (selected clip)     │
│ Transitions |     │  │         canvas           │   │ ▸ Transform         │
│ Effects | Markers │  └──────────────────────────┘   │ ▸ Opacity & Blend   │
│                   │  00:00:12:04 / 00:03:40:00      │ ▸ Crop              │
│  [grid of media]  │  ⏮ ◀ ▶/⏸ ▶ ⏭   quality ▾  ⛶    │ ▸ Speed             │
│                   │                                 │ ▸ Audio             │
│                   │                                 │ ▸ Effects (stack)   │
├───────────────────┴────────────────────────────────┴─────────────────────┤
│ Toolbar: ⬚ Select(V) ✂ Razor(C) ⇔ Ripple(B) ⇹ Roll(R) | 🧲 Snap(N) | ─●── zoom│
├──────────────────────────────────────────────────────────────────────────┤
│ Track headers │ ruler 00:00 ── 00:05 ── 00:10 ──▼ playhead               │
│ V2 👁 🔒       │        [ title ]                                         │
│ V1 👁 🔒       │ [ clip A        ][ clip B      ][ clip C ]               │
│ A1 🔇 S 🔒     │ [~~~ waveform ~~][~~~~~~~~~~~~~]                          │
│ A2 🔇 S 🔒     │ [~~~~~~~~ music ~~~~~~~~~~~~~~~~~~~~~~~~]                 │
└──────────────────────────────────────────────────────────────────────────┘
```

- Panels resizable via splitters; sizes persisted in settings.
- Minimum window 1280×720; remember window size/position/maximized.
- Dark theme default (neutral grays, one accent color); light theme optional.
- Empty states: "Drop media here" in bin; "Drag clips here to start" on empty timeline.

### 16.2 Timeline rendering (Canvas 2D)
- One `<canvas>` for the track area (device-pixel-ratio aware), React only for headers, toolbar and overlays.
- Viewport: `pixelsPerFrame` (zoom), `scrollX`, `scrollY`. Only visible clips are drawn (binary search on sorted clip starts per track).
- Draw order: track backgrounds → gaps highlight → clips (body, thumbnails/waveform, name, effect badge `fx`, speed badge, keyframe diamonds, fade handles) → transitions → markers → In/Out shading → snap line → marquee → playhead.
- Redraw on state change via `requestAnimationFrame` coalescing; target 60 fps with 1,000 clips.
- Thumbnails: draw from sprite images; waveforms: choose peak level matching zoom.

### 16.3 Timeline interactions
Implemented as small state machines in `renderer/timeline/interactions/`:
- **Hit testing** (`hitTest.ts`): returns `{ type: 'clipBody' | 'clipEdgeStart' | 'clipEdgeEnd' | 'fadeHandle' | 'transition' | 'transitionEdge' | 'keyframe' | 'marker' | 'gap' | 'empty' | 'ruler', … }`; edge zone 6 px.
- **Drag clip:** ghost preview, snap, target-track highlight; commit as one transaction; `Esc` cancels.
- **Trim:** shows a two-up frame preview in the player (outgoing/incoming frame) and a delta tooltip (`+00:00:01:12`).
- **Snapping** (`shared/timeline/snapping.ts`): candidates = clip edges on all tracks, playhead, markers, In/Out; threshold 8 px converted to frames; nearest wins; hold `Alt` to temporarily disable.
- **Locked tracks** reject edits with a subtle shake/tooltip.

### 16.4 Media bin
Grid cards: thumbnail, name, duration, type icon, status badge. Right-click: Reveal in Explorer, Rename, Replace/Relink, Generate proxy, Properties, Remove.

### 16.5 Inspector
- Collapsible sections; each property row: label, value control (NumberScrubber, slider, color picker), keyframe stopwatch, reset.
- Multi-selection: shows common properties; edits apply to all (one undo entry).
- Transition selected → transition params. Nothing selected → project info.

### 16.6 Dialogs
New Project, Export, Settings, Keyboard Shortcuts, Relink Media, Recovery, About, Confirm (unsaved changes / delete used media), AI consent download.

### 16.7 Menus
- **File:** New, Open, Open Recent ▸, Save, Save As, Import Media, Export, Collect Project, Project Settings, Exit
- **Edit:** Undo, Redo, Cut, Copy, Paste, Paste Attributes, Duplicate, Delete, Ripple Delete, Select All, Deselect, Preferences
- **Clip:** Split, Speed/Duration, Reverse, Detach Audio, Link/Unlink, Freeze Frame, Enable/Disable, Label ▸, Add Default Transition
- **Timeline:** Add Track ▸, Delete Track, Close Gaps, Add Marker, Set In/Out, Clear In/Out, Snapping, Zoom In/Out/Fit
- **View:** Preview Quality ▸, Use Proxies, Safe Areas, Fullscreen Preview, Theme ▸
- **Window:** panels toggles, Reset Layout
- **Help:** Keyboard Shortcuts, Open Logs Folder, About

### 16.8 Settings
General (autosave interval, recent count, theme, language placeholder), Playback (auto proxy for > 1080p, adaptive quality, audio scrubbing), Export (default folder, hardware encoding), Cache (location, size, clear), Shortcuts, Advanced (history limit, job concurrency, enable debug overlay showing fps/dropped frames).

---

## 17. Keyboard shortcuts (defaults, rebindable)

| Action | Shortcut |
|---|---|
| Play / pause | Space |
| Shuttle reverse / pause / forward | J / K / L |
| Previous / next frame | ← / → |
| Back / forward 10 frames | Shift+← / Shift+→ |
| Previous / next edit point | ↑ / ↓ |
| Go to start / end | Home / End |
| Split at playhead (selected or targeted tracks) | S, Ctrl+K |
| Split all tracks | Shift+S |
| Tools: Select / Razor / Ripple / Roll / Slip / Slide | V / C / B / R / Y / U |
| Delete / Ripple delete | Delete, Backspace / Shift+Delete |
| Undo / Redo | Ctrl+Z / Ctrl+Y, Ctrl+Shift+Z |
| Cut / Copy / Paste / Duplicate | Ctrl+X / Ctrl+C / Ctrl+V / Ctrl+D |
| Paste attributes | Ctrl+Alt+V |
| Select all / Deselect | Ctrl+A / Ctrl+Shift+A |
| Select forward from playhead | A |
| Link / unlink | Ctrl+L |
| Snapping | N |
| Add marker / next / previous marker | M / Shift+M / Ctrl+Shift+M |
| Set In / Out / Clear | I / O / Alt+X |
| Insert / Overwrite from source | , / . |
| Zoom in / out / fit | = / - / \ |
| Add default transition | Ctrl+T |
| Add text | T |
| New / Open / Save / Save As | Ctrl+N / Ctrl+O / Ctrl+S / Ctrl+Shift+S |
| Import / Export | Ctrl+I / Ctrl+E |
| Fullscreen preview | F |
| Nudge clip 1 frame | Alt+← / Alt+→ |

Shortcuts are disabled while typing in a text field. Keymap lives in `renderer/shortcuts/keymap.ts` as `{ commandId, keys[] }` and dispatches the same commands as menu items.

---

## 18. Non-functional requirements

| Area | Requirement |
|---|---|
| Startup | Window visible < 3 s on a typical laptop (SSD, 8 GB RAM) |
| Preview | 1080p30 timeline with 2 video tracks + text + 2 audio tracks plays without dropped frames on integrated graphics (with proxies where needed); 4K via proxies |
| Timeline UI | 60 fps drag/zoom with 1,000 clips |
| Scrub latency | Paused seek to new frame shows image in < 150 ms for proxy/H.264 media |
| Export speed | 1080p H.264 export ≥ 1× real time on a mid-range PC with hardware encoding |
| Accuracy | Frame-accurate cuts; exported duration = timeline duration ±1 frame; A/V sync within 1 frame over 60 minutes |
| Memory | < 2 GB RSS for a 30-minute 1080p project with 300 clips |
| Robustness | Corrupt/unsupported files never crash the app; FFmpeg errors are reported clearly |
| Data safety | Atomic saves, backup file, autosave, recovery; source media never modified |
| Paths | Unicode, spaces, long paths (`\\?\`) supported; FFmpeg always called with argument arrays via `spawn`, never via a shell string |
| Security | Section 4 rules; no telemetry; no network calls except explicit AI model download |
| Accessibility | All actions reachable via menus and keyboard; visible focus; tooltips |
| Hardware | Windows 10 1809+ / 11, x64, WebGL2-capable GPU, 8 GB RAM recommended |

---

## 19. Error handling, logging, testing

### 19.1 Errors
- Typed errors (`MediaProbeError`, `FFmpegError { code, stderrTail }`, `ProjectLoadError`, `ExportError`) cross IPC as `{ code, message, details }`.
- User-facing messages are plain language with a suggested action ("This file uses a codec we can't read. Try converting it to MP4.").
- Renderer has a top-level React error boundary that offers "Save a copy and reload".
- `process.on('uncaughtException')` in main logs and shows a dialog, then attempts an emergency autosave.

### 19.2 Logging
`electron-log` at info level by default, debug in dev. Log every FFmpeg command line (paths included) and the last 50 lines of stderr on failure. Rotate at 5 MB, keep 5 files.

### 19.3 Testing
- **Unit (Vitest)**, required for: `time.ts`, `timeline/operations.ts`, `snapping.ts`, `keyframes.ts`, `evaluateFrame.ts`, `evaluateAudio.ts`, `audioGraph.ts`, `encoderArgs.ts`, project migrations, history (undo/redo/transactions). Target ≥ 90% coverage of `src/shared`.
- **Fixtures:** `scripts/make-fixtures.ts` generates test media with FFmpeg (`testsrc2`, `sine`, `color`, numbered-frame overlays via `drawtext` on frame numbers, VFR sample, rotated phone sample, 4K sample, audio-only, PNG with alpha). No large binaries committed.
- **Golden-frame tests:** render chosen frames of fixture projects via the export pipeline and compare against reference PNGs (per-pixel tolerance, SSIM > 0.99). Also compare preview `snapshotFrame` vs export frame for the same project.
- **E2E (Playwright + Electron):** launch app → new project → import fixtures → place on timeline → split → undo/redo → save → reopen → export → FFprobe asserts duration, resolution, streams.
- **Frame-accuracy test:** fixture with burned-in frame numbers; cut at frame 100; exported frame 0 of the second clip must show the expected number.
- CI-ready scripts: `npm run lint`, `npm run typecheck`, `npm test`, `npm run test:e2e`.

---

## 20. Build phases and acceptance criteria

Every phase ends with: lint + typecheck + tests green, `npm run dist` succeeds, the **portable .exe is launched and manually checked**, `PROGRESS.md` updated, git commit/tag `phase-N`.

### Phase 0 — Foundation & packaging
- Scaffold electron-vite + React + TS + Tailwind + Zustand + Vitest + Playwright + ESLint/Prettier.
- FFmpeg/FFprobe path resolution in dev and packaged builds; log versions at startup.
- `media://` protocol with Range support.
- Typed preload bridge skeleton; logger; settings service.
- electron-builder: NSIS installer + portable, icon, file association `.cfproj`, single-instance lock.
- Empty app shell with resizable panel layout and menu bar; About dialog.
- **Acceptance:** double-click portable `.exe` → window opens → About shows app + FFmpeg versions; a `.cfproj` double-click focuses the running app.

### Phase 1 — MVP: cut, preview, export
- Time model, data model, Zod schemas, project IO (atomic save), start screen, new project dialog.
- Import + probe + bin thumbnails + mandatory proxies for non-previewable media; job queue with progress UI.
- Timeline canvas: tracks, clips, ruler, playhead, zoom/scroll, drag/move, trim, split, delete/ripple delete, snapping.
- Undo/redo with transactions.
- `evaluateFrame`, compositor (layer + blend passes, fit modes), preview playback engine with audio clock, volume.
- Export engine: audio plan → WAV, export window, DecoderPool, compositor render loop, libx264 encoding, progress, cancel, verification.
- Features: P-1–P-4, P-9, M-1–M-4, M-6, M-10 (mandatory), T-1–T-9, V-1–V-3, V-5, V-6, V-12, C-7, A-1, O-1, O-2, O-5, O-11, U-1, U-2, U-6–U-8, U-10.
- **Acceptance:** import two MP4s (one MKV), an MP3 and a PNG; arrange on tracks; trim, split, ripple delete; undo/redo every step; save, quit, reopen; export 1080p MP4 that plays in Windows Media Player; duration within ±1 frame; frame-accuracy test passes.

### Phase 2 — Usable editor
- Features: P-5, P-6, M-5, M-12, T-10–T-14, T-17, T-22, T-23, V-4, V-8, V-13, C-1–C-5, C-8, C-9, C-11 (reset), C-12, X-1–X-3, A-2, A-4, O-4, O-8, U-4, U-5, U-11.
- **Acceptance:** build a 60-second 1080×1920 short: background clip scaled to Fill, picture-in-picture overlay positioned with canvas handles, a title with stroke and shadow, music with fade-out and a voice clip; export with the Reel preset; preview and export frames match (golden test).

### Phase 3 — Creative tools
- Features: P-7, M-7–M-9, M-10 (optional proxies), M-11, T-15, T-18–T-21, V-7, V-9–V-11, V-14, C-6, C-11 (paste attributes), K-1–K-3, X-4, R-1–R-6, E-1–E-3, A-3, O-3, O-6, O-7, U-3.
- **Acceptance:** 4K footage edits smoothly via proxies; keyframed zoom + cross dissolve + color correction look identical in preview and export; hardware encoder is used when present and falls back cleanly when disabled.

### Phase 4 — Pro features
- Features: P-8, T-16, C-10, K-4–K-6, X-5, X-6, E-4, E-5, E-10, A-5, A-6, A-9, O-9, U-9.
- **Acceptance:** green-screen clip keyed over a background with LUT applied; voiceover recorded into the timeline; three exports queued and completed.

### Phase 5 — Advanced & AI
- Features: E-6–E-9, A-7, A-8, O-10, AI-1–AI-4.
- **Acceptance:** 5-minute talking-head video: silence removal applied in one undoable step, auto-captions generated, styled and burned in, `.srt` exported.

### Phase 6 — Stretch
- Features: T-24, AI-5–AI-7, auto-update (`electron-updater`, opt-in), code signing setup docs.

---

## 21. Packaging and distribution

`electron-builder.yml` essentials:

```yaml
appId: com.clipforge.app
productName: ClipForge
directories: { output: release, buildResources: build }
files: [ "out/**" ]
asarUnpack: [ "**/node_modules/ffmpeg-static/**", "**/node_modules/ffprobe-static/**" ]
extraResources:
  - from: resources/bin
    to: bin
    filter: [ "**/*" ]
win:
  target:
    - { target: nsis, arch: [ x64 ] }
    - { target: portable, arch: [ x64 ] }
  icon: build/icon.ico
  fileAssociations:
    - { ext: cfproj, name: ClipForge Project, description: ClipForge Project, icon: build/icon.ico }
nsis:
  oneClick: false
  allowToChangeInstallationDirectory: true
  createDesktopShortcut: true
  createStartMenuShortcut: true
  artifactName: ClipForge-Setup-${version}.exe
portable:
  artifactName: ClipForge-${version}-portable.exe
```

Scripts in `package.json`:
- `dev` — run with hot reload
- `build` — compile all targets
- `dist` — build + package to `release/`
- `lint`, `typecheck`, `test`, `test:e2e`, `fixtures`

Notes:
- The app is unsigned by default; Windows SmartScreen will warn on first launch ("More info → Run anyway"). Code signing is a Phase 6 item.
- FFmpeg licence: `ffmpeg-static` ships a GPL build, fine for personal use. For public distribution switch to an LGPL build and show the licence notice (About dialog already shows it).
- Portable build stores settings in `%APPDATA%/ClipForge` like the installed build.

---

## 22. Working rules for Claude Code

1. **Read this spec first**, then write a plan for the current phase in `PROGRESS.md` before coding.
2. **One phase at a time.** Do not build later-phase features early, but do design interfaces (types, registries, IPC) so later phases fit without rewrites.
3. **Pure logic in `src/shared`** with unit tests written alongside it; UI code stays thin.
4. **Commit often** with feature IDs: `feat(T-5): split clip at playhead`, `fix(V-12): resync audio after seek`.
5. **Never** call FFmpeg through a shell string; use `spawn(path, argsArray, { windowsHide: true })`.
6. **Preview/export parity is sacred.** Any visual feature goes through `evaluateFrame` + compositor. Never implement a visual effect only in FFmpeg filters for export.
7. **All project changes go through `projectStore.apply`** so undo/redo always works.
8. **Verify packaged builds**, not just dev mode: FFmpeg paths, `media://` protocol and the export window must work in the `.exe`.
9. Record ambiguous decisions in `docs/DECISIONS.md`; record known bugs and remaining work in `PROGRESS.md`.
10. If a requirement is technically impractical, stop and explain the trade-off with a proposed alternative instead of silently dropping it.
11. Keep dependencies minimal and well-maintained; justify each new one in `docs/DECISIONS.md`.
12. At the end of each phase, report: features completed (IDs), tests status, `.exe` paths, known issues, and what to manually check.

---

## 23. Glossary

| Term | Meaning |
|---|---|
| Clip | An instance of media (or text/solid) placed on the timeline |
| Source in | The point in the media file where a clip starts |
| Handles | Unused source media before/after a clip's in/out, needed for transitions |
| Ripple | An edit that shifts later clips to close or open space |
| Roll | Moving a cut point between two clips, trimming one and extending the other |
| Slip | Changing a clip's source in/out without moving it on the timeline |
| Slide | Moving a clip between its neighbors, trimming them to compensate |
| Proxy | A lightweight copy of media used only for smooth preview |
| Compositor | The WebGL renderer that combines layers into a frame |
| RenderGraph | The resolved description of everything visible at one frame |
| Linked clips | Video and audio clips from the same file that move/trim together |
| Transaction | A group of state changes recorded as one undo step |

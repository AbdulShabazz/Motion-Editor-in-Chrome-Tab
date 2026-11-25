# Motion Editor

A browser-based animation editor for two interacting wrestlers. It implements the features
specified in `Wrestling_animation_tools.md`: one shared timeline for both characters, timed
grips and mat pins, body follow, hand shaping, visible errors, deterministic curves, Quick
Load with step-wise playback, and baked export for Unreal Engine.

## Opening the editor

Open `MotionEditor.html` in Chrome or Edge (double-click; no server and no installation).
The file is self-contained and works offline. Work is saved automatically in the browser;
use **Save** for a portable `.motion.json` project file.

On first start the built-in *Arm Drag* example is loaded so that every panel has content.

## Tabs

| Tab | Purpose |
| --- | --- |
| **Pose** | Pose library, paired presets for both wrestlers, Pose Strength, Mirror, Copy to other. |
| **Animate** | Main workspace: viewport, inspector, timeline with keys and contact bars. |
| **Curves** | Value curves per channel, key dragging, timing (ease) curve per key, spline or linear paths. |
| **Contacts** | Grips and mat pins: attach and release times, blend, orientation hold, solve priority. |
| **Rig** | Import a character (FBX / GLB), bone mapping, joint limits, rig profile JSON. |
| **Bake & Export** | Validation report, bake, exports, starting offsets, performance measurements. |

## Feature map (specification to editor)

| Specification | Where |
| --- | --- |
| Both wrestlers on one timeline | Timeline rows per wrestler; every frame solves both. |
| Timed grip constraints | Contacts tab, or *Grip* buttons; bar on the timeline; offset and orientation relative to the opponent's bone. |
| Mat contact pins (feet, knees, elbows, hands) | Inspector > Contact Pins toggles, or Contacts tab. |
| Contact orientation | *Hold orientation* per contact; limited to the wrist range, with a warning when it cannot be met. |
| Paired pose presets | Pose tab > Paired presets (Neutral Stance, Collar Tie, Shot vs Sprawl, Reach and Brace); save your own. |
| Visible errors | Red markers in the viewport, ruler marks on the timeline, Inspector > Visible Errors, validation report: unreachable targets, joint limits, body intersections, contacts that cannot be held. |
| Hand Openness spinner | Inspector > Hand: dial plus slider and numeric field, 0 to 100 %; Grip Shape (fist, hook, cupped, relaxed, spread), Finger Spread, Thumb Opposition, Individual Finger Curl, Wrist Flex / Deviation / Twist. |
| Body Follow | Inspector > Body Follow: driving target, Body Follow, Torso Follow, Reach Threshold, Path Smoothness, Follow Lag, Allowed Movement. Contact pins take priority over follow. |
| Root Travel | Separate Root Position and Root Yaw channels; body follow never changes them. |
| Other controls | Elbow Swivel, Knee Swivel, Shoulder Raise, Pelvis Height, Torso Lean / Turn, Foot Roll / Heel Lift, Pose Strength, Mirror Pose. |
| Numeric value, drag, reset, key | Every continuous control has a numeric field, a slider or gizmo, a reset button and a key button. |
| Deterministic curves | A frame always evaluates to the same pose, in any scrub order (covered by a test). |
| Spatial path separate from timing | Path interpolation (spline / linear) per channel; timing curve per key (cubic bezier presets or custom). |
| Worker architecture | Solver in a persistent Web Worker, bake in a second worker, rendering on the GPU (WebGL 2). Requests are coalesced, stale results are rejected, drags solve at draft quality and refine on release. |
| Measurements | Bake & Export > Performance: input to visible pose, solver time, render time, bake duration. |
| Quick Load, step-wise playback | Animations panel: Quick Load, Open Copy, Insert, Import, Connect folder. Transport: Continuous / Step, Auto Step, Pause while editing. |
| Unreal assets | Rig > Import character (FBX or GLB exported from Unreal). The skeleton, its axes and its units are kept for export. |
| Baking and export | FBX and GLB per wrestler, baked JSON, contact metadata (JSON and CSV), starting offsets, project file, helper scripts, one ZIP bundle. |

## Keyboard

| Key | Action |
| --- | --- |
| Space | Play / pause |
| Left / Right (Shift: 10 frames) | Step frames |
| Home / End | First / last frame |
| `,` / `.` | Previous / next key |
| K / Shift+K | Key selected control / key whole pose |
| Delete | Delete selected keys |
| Ctrl+C / Ctrl+V | Copy / paste keys at the playhead |
| Ctrl+Z / Ctrl+Y | Undo / redo |
| Ctrl+S | Save project file |
| W / E | Move / rotate gizmo |
| F | Frame both wrestlers |
| 1 to 6 | Switch tab |
| Esc | Cancel a drag in progress |

## Unreal workflow

1. In Unreal, export the wrestler Skeletal Mesh as FBX (or run `tools/unreal_export_for_editor.py`
   on the selected assets).
2. In the editor: **Rig > Import character** (or *Import for both*). Check the bone mapping.
   The character is cached in the browser, so later sessions only load clips.
3. Optional: **Animations > Import** converts existing animations (FBX / GLB on the same
   skeleton) to editable control keys. Body follow is switched off for converted clips,
   because the imported motion already contains the whole body.
4. Animate. Use **Bake & Export > Validate and bake**, then **Export bundle (.zip)**.
5. Unzip the bundle into a folder and run `unreal_import_animations.py` in Unreal after
   editing its three settings (folder, destination path, skeleton assets). It imports the
   FBX clips, enables root motion, and adds contact notifies and curves from
   `motion_metadata.json`.

Each clip starts at its own actor origin by default (*Root space: Own actor origin*). Place
the two actors with the `startOffset` values in `motion_metadata.json`, which are given in
editor axes and in Unreal axes.

## Verification status

Verified here, in headless Chromium, by the tests in `test/` (58 browser checks, plus the
solver tests):

- solver: IK reach, bone lengths, mirror symmetry, pins, grips, body follow, determinism,
  IK bones following their targets, every built-in pose and clip validating cleanly;
- interaction: sliders, dial, gizmo, timeline and curve editing, shortcuts, undo, step mode;
- FBX export read back with an independent FBX parser (three.js FBXLoader): bone error
  below 0.001 cm, for Y-up and for Z-up skeletons;
- GLB export re-imported as a character, and its animation converted back to control keys.

Not verified, because no Unreal Engine, Blender or project asset was available:

- import of the exported FBX in Unreal Engine on your wrestler skeleton;
- the three scripts in `tools/` (written against the Unreal 5 and Blender Python APIs,
  never executed);
- import of your actual wrestler mesh. The mapping covers Unreal mannequin and Mixamo
  bone names; other naming needs manual mapping in the Rig tab.

Known limits: converting a foreign animation to control keys is an approximation of the
editor's control rig (hands, feet, pelvis, elbows and knees are matched; spine curvature
and clavicle motion are simplified). Twist and corrective bones follow their parent bone.
The built-in clips are blocking-quality examples, not finished animation.

## Building from source

```
npm install
npm run build        # writes dist/MotionEditor.html (single file, minified)
npm test             # solver tests (Node)
npm run test:browser # browser tests; needs Playwright with Chromium
```

Source layout: `src/core` (math, channels, curves, rig, solver, presets; no DOM, runs in the
worker), `src/worker`, `src/app` (state, solver client, transport, clips, models),
`src/view` (three.js viewport and characters), `src/ui` (panels), `src/io` (FBX, GLB, ZIP,
clip import), `tools` (Unreal and Blender scripts), `test`.

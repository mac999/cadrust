# PRD: cadrust — a Fusion-style parametric modeler on OpenCascade, in Rust, for the browser and the command line

Status: v0.6, 2026-10-18. Feature dependency graph with content-keyed incremental evaluation (crates/cadrust-core/src/graph.rs): only the subgraph downstream of an edit is recomputed, undo replays from the cache; one cache per server / wasm worker. Owner: Taewook Kang. Phases 0-5 implemented (see README.md): modeling, sketch constraints, UX polish, add-ins, serverless WASM build, persistent naming by face history, v1 extras (extrude extents and taper, sphere / torus, scale, split body, offset face / press pull, delete, STEP import as a feature, booleans on every body maker, orthographic view). Open: Draft (no draft-angle call in cadrum yet; taper is done by lofting to the 2D-offset profile), WIT/component add-in API, STEP metadata via ruststep, surface tools.

## 0. One-paragraph summary

cadrust is a parametric solid modeler with a Fusion 360-like UX that runs in the browser, plus a headless
CLI that exposes the same kernel as a pipeline tool. The geometry kernel is Open CASCADE Technology
(OCCT) 8.0 reached through the `cadrum` Rust crate, which links OCCT statically and compiles to both
native and WebAssembly. Everything that is not pixels is Rust: document model, feature history, sketch
constraint solver, STEP I/O, command bus, CLI. The web view is TypeScript + three.js driving the Rust core
in a Web Worker. The design is MVC with the Model crate shared verbatim between the web app and the CLI.
STEP goes through OCCT's own reader/writer; a pure-Rust ISO 10303-21 parser (ruststep) is used only for
metadata, validation and the no-OCCT CLI path. Add-ins are WebAssembly components with a WIT-defined API,
plus TypeScript UI add-ins. UI default language is English with Korean as the first additional locale. All
code comments, logs and messages are English. Scope is deliberately "80 % of daily Fusion modeling": sketch → extrude/revolve/sweep/loft → fillet/chamfer/
shell/boolean/pattern → STEP out. No assemblies, CAM, simulation or cloud collaboration in v1.

## 1. Goals and non-goals

### Goals
1. Fusion-familiar modeling: a Fusion user should find the sketch, create, modify and construct tools where
   they expect them, with the same keyboard shortcuts (E, Q, L, C, D, F, M, S).
2. Parametric history (timeline) with edit-in-place and reliable re-evaluation.
3. STEP AP214/AP242 import and export with names and colours.
4. Runs fully client-side in the browser; no upload of design data. Also runs as a native CLI that executes
   the same document format as a pipeline (`cadrust run pipeline.json`).
5. Korean/English UI switch at runtime, dark and light themes, modern flat UI.
6. Cost-effective: lean on OCCT, ezpz and three.js; write only the glue, the document model and the UX.
7. Modular crates with a kernel trait so the OCCT backend can later be swapped or augmented (e.g. truck).
8. Add-in system: third parties add commands, features, importers/exporters and UI panels without forking.
9. English-first: UI strings, code comments, logs, errors and docs are English and concise; Korean is a locale.

### Non-goals (v1)
Assemblies and joints, CAM, FEA, sheet metal, T-splines/sculpt, rendering/animation, multi-user editing,
cloud storage, mesh editing beyond import/visualise, drawings (2D sheets), PDM.

## 2. Users and jobs

| user | job | success signal |
|---|---|---|
| Mechanical/architectural designer who knows Fusion | model a bracket, enclosure, fixture with holes and fillets, export STEP | finishes a 20-feature part without reading docs |
| Developer integrating CAD into a pipeline | generate parametric parts from JSON, convert STEP→glTF, validate geometry in CI | one CLI call per step, exit codes, no GUI |
| Reviewer (interview, portfolio) | read code and see a working demo | clean crate boundaries, tests, measured numbers |

## 3. Landscape (what exists, what we reuse)

| project | kernel | stack | licence | relevance |
|---|---|---|---|---|
| Fusion 360 | Autodesk ShapeManager | desktop, cloud | proprietary | UX reference only |
| Onshape | Parasolid | browser, server-side kernel | proprietary | UX reference, server model we avoid |
| FreeCAD 1.0 | OCCT | C++/Python, Qt | LGPL-2.1 | feature semantics, topological naming lessons |
| CADmium | truck (pure Rust) | Rust→WASM, SvelteKit, three.js | Elastic 2.0 (not OSI) | closest architecture precedent; no fillet/STEP import, licence not reusable |
| Chili3D | OCCT→WASM | TypeScript | AGPL-3.0 | proves OCCT-in-browser works; AGPL means we read, not copy |
| opencascade.js | OCCT→WASM (emscripten) | JS | LGPL-2.1 | fallback if cadrum WASM has gaps |
| cadrum (crates.io) | OCCT 8.0 static | Rust, native + wasm32 | check crate licence (MIT/Apache expected), OCCT LGPL-2.1+exception | **chosen kernel binding** |
| opencascade-rs (bschwind / mkovaxx fork) | OCCT 7.x via cxx | Rust native | LGPL for OCCT, binding MIT/Apache | fallback native binding, work in progress |
| truck | pure Rust B-rep/NURBS | Rust | Apache-2.0 | future second backend behind the kernel trait |
| ezpz (kcl-ezpz, Zoo) | 2D constraint solver | Rust | MIT (announced; verify LICENSE file) | **chosen sketch solver** |
| SolveSpace / libslvs | 2D/3D solver | C++ | GPL-3.0 | avoid in product code |
| planegcs (FreeCAD) | 2D solver | C++ | LGPL-2.1 | fallback via dynamic link if ezpz lacks a constraint |
| ruststep (ricosjp) | ISO 10303-21 parser + EXPRESS compiler (espr) | Rust | MIT/Apache-2.0 | **chosen for STEP metadata/validation**; not a geometry kernel |
| STEPcode (stepcode.org) | EXPRESS/Part 21 C++ | C++ | BSD-3 | reference only |
| three.js | renderer | TS | MIT | chosen view renderer |
| CadQuery / build123d | OCCT scripting | Python | Apache-2.0 | reference for feature → OCCT call mapping |

Licensing position: ship under MIT for our code; OCCT is LGPL-2.1 with the OCCT exception. For the WASM
bundle (static link by nature) we comply by publishing the exact OCCT version and build flags, keeping
OCCT unmodified, and documenting the attribution. The exception explicitly covers header material in our
object code; for the statically linked OCCT libraries we additionally provide relink instructions (open
source project, so trivially satisfied). Anything GPL/AGPL (SolveSpace, Chili3D, plyfile) stays out of the
dependency tree.

## 4. Architecture

### 4.1 Crates and MVC mapping

```
cadrust/
  crates/
    cadrust-kernel     Model   GeometryKernel trait + OcctKernel (cadrum). Shapes, booleans, fillets, tessellation, mass props.
    cadrust-sketch     Model   2D entities, constraints, dimensions; ezpz solver adapter; profile extraction (loops → faces).
    cadrust-doc        Model   Document, Body, Feature enum, Timeline, Parameters (expressions), persistent naming, undo/redo.
    cadrust-io         Model   STEP/IGES via kernel, glTF/OBJ/STL export, .cadrust JSON (serde) read/write, DXF/SVG sketch import.
    cadrust-cmd        Controller  Command bus: typed commands, validation, transaction + undo, event stream to views.
    cadrust-addin      Controller  Add-in host: WIT interface, wasmtime (native) / browser WebAssembly loader, manifest, permissions.
    cadrust-wasm       Controller  wasm-bindgen facade of cadrust-cmd for the browser (runs in a Web Worker).
    cadrust-cli        Controller  Headless CLI; pipeline runner; same commands as the UI.
    cadrust-server     Controller (optional) axum HTTP/WebSocket for large models or shared sessions. Not in v1 critical path.
  web/              View    TypeScript, React, three.js, Tailwind. Talks only to cadrust-wasm through a message protocol.
```

Rules: `web/` never touches geometry; it renders meshes and sends commands. `cadrust-doc` never imports
three.js types or DOM. `cadrust-kernel` is the only crate that knows OCCT. The CLI and the WASM facade are
thin; all logic lives in `cadrust-cmd` and below so the two front ends cannot diverge.

### 4.2 Data flow

```
user gesture → web view → Command (JSON) → Worker (cadrust-wasm) → cadrust-cmd.execute()
   → cadrust-doc mutates timeline → cadrust-kernel rebuilds affected features
   → tessellation (positions, normals, edges, face ids) → transferable ArrayBuffers → three.js
   → selection hit test on GPU id buffer → face/edge/vertex ids → next Command
```

Re-evaluation is incremental: a feature is rebuilt only if it or any upstream dependency changed
(dependency graph in `cadrust-doc`). Tessellation runs per body with a deflection tied to bounding-box size
and caches by shape hash.

### 4.3 Document format (.cadrust, JSON)

```json
{
  "version": 1,
  "units": "mm",
  "parameters": {"w": "40 mm", "h": "w/2 + 3"},
  "timeline": [
    {"id": "sk1", "type": "Sketch", "plane": {"ref": "XY"},
     "entities": [{"id":"l1","type":"Line","p0":[0,0],"p1":["w",0]}, ...],
     "constraints": [{"type":"Horizontal","e":"l1"}, {"type":"Distance","a":"p0","b":"p1","value":"w"}]},
    {"id": "ex1", "type": "Extrude", "profiles": ["sk1:loop0"], "distance": "h", "op": "NewBody"},
    {"id": "fi1", "type": "Fillet", "edges": [{"ref": "ex1/edge[top,+x]"}], "radius": "2 mm"}
  ]
}
```

The same file is a pipeline for the CLI. Expressions are evaluated by a small `evalexpr`-based engine
with units. References use persistent names (see 5.9), never raw OCCT indices.

### 4.4 Message protocol (view ↔ worker)

Commands: `ApplyFeature`, `EditFeature`, `Reorder`, `Suppress`, `SetParameter`, `Select`, `Undo`, `Redo`,
`Import`, `Export`, `Tessellate(quality)`. Events: `DocumentChanged(diff)`, `MeshUpdated(body, buffers)`,
`FeatureError(id, msg)`, `Progress`. All messages are serde JSON except mesh buffers, which are raw
`Float32Array`/`Uint32Array` transferred without copy.

## 5. Functional spec: menus and algorithms

Priority: P0 = MVP, P1 = v1, P2 = later. OCCT calls are the ones cadrum must expose; gaps are extended
through cadrum's cxx wrapper (fork + PR) or a sibling `cadrust-occt-ext` crate.

### 5.1 File
| item | P | implementation |
|---|---|---|
| New / Open / Save / Save As (.cadrust) | P0 | serde JSON; browser File System Access API with download fallback; CLI path |
| Import STEP (AP203/214/242) | P0 | `STEPCAFControl_Reader` → XCAF for names/colours; one body per solid |
| Export STEP AP214 (AP242 option) | P0 | `STEPCAFControl_Writer`, write names and colours |
| Export glTF / OBJ / STL | P0 | from tessellation; glTF via `gltf` crate |
| Import IGES, BREP | P1 | `IGESControl_Reader`, `BRepTools::Read` |
| Import DXF/SVG into sketch | P2 | `dxf` crate, `usvg` |
| Recent files, autosave to IndexedDB | P1 | |

### 5.2 View
| item | P | implementation |
|---|---|---|
| Orbit / pan / zoom, Fusion mouse mapping (MMB pan, Shift+MMB orbit, wheel zoom) | P0 | three.js custom controls |
| ViewCube, named views (top/front/right/iso), fit, look-at | P0 | three.js gizmo |
| Display: shaded, shaded+edges, wireframe | P0 | edge buffer from `BRepMesh` polygons on triangulation |
| Grid, snap settings, units | P0 | |
| Section analysis (clip plane) | P1 | three.js clipping planes; capping optional |
| Visual style: dark/light, theme tokens | P0 | CSS variables, persisted per user |
| Perspective/orthographic toggle | P1 | |

### 5.3 Sketch
Entities (P0): line, rectangle (2-point, center), circle (center, 3-point), arc (3-point, center),
polygon, spline (P1), point, slot (P1), text (P2).
Operations (P0): trim, extend, offset, mirror, fillet, project edge/face into sketch; (P1) intersect,
pattern in sketch.
Constraints (P0): coincident, horizontal, vertical, parallel, perpendicular, tangent, equal, fix,
midpoint, concentric, symmetric; dimensions: distance, radius/diameter, angle. (P1) collinear, curvature.

Solver spec:
- Variables = entity coordinates; each constraint = residual function(s). ezpz solves the sparse nonlinear
  least-squares system (Levenberg-Marquardt / Gauss-Newton; details are the crate's). Degrees-of-freedom
  display = variables − rank(Jacobian), shown per sketch as Fusion does (blue = unconstrained, black = fully).
- Interaction: dragging a point adds a temporary "soft" target with low weight so the sketch follows the
  cursor while hard constraints hold.
- Over-constraint: detect by rank deficiency; refuse the new constraint with a highlight.
- Profile extraction: closed loops via planar graph face enumeration (half-edge, angle-sorted), nested loops
  become holes; each loop → `BRepBuilderAPI_MakeWire` → `MakeFace`.
- Fallback if ezpz lacks a constraint: implement residual ourselves in the adapter; last resort planegcs
  via dynamic link (LGPL) on native only.

### 5.4 Create (modeling)
| feature | P | OCCT | notes |
|---|---|---|---|
| Extrude (distance, to object, symmetric, two sides, taper) | P0 | `BRepPrimAPI_MakePrism`, taper via `BRepOffsetAPI_DraftAngle` or loft | op: New body / Join / Cut / Intersect via `BRepAlgoAPI_*` |
| Revolve (angle, full) | P0 | `BRepPrimAPI_MakeRevol` | axis from sketch line or construction axis |
| Sweep (path, guide rail P2) | P1 | `BRepOffsetAPI_MakePipeShell` | Frenet vs fixed binormal modes |
| Loft (profiles, rails P2) | P1 | `BRepOffsetAPI_ThruSections` (solid=true, ruled option) | |
| Hole (simple, counterbore, countersink) | P1 | cylinder/cone built with `MakeCylinder`/`MakeCone`, then Cut | thread cosmetic only |
| Primitives: box, cylinder, sphere, torus | P0 | `BRepPrimAPI_Make*` | |
| Pipe (circular profile along path) | P2 | `MakePipeShell` | |
| Rectangular / circular pattern (features or bodies) | P1 | transform copies + Fuse | compute-cost note: pattern of a feature re-runs the feature |
| Mirror | P1 | `BRepBuilderAPI_Transform` with gp_Trsf mirror + Fuse | |
| Thicken (surface → solid) | P2 | `BRepOffsetAPI_MakeThickSolid` | |
| Boundary fill | P2 | `BOPAlgo_MakerVolume` | |

### 5.5 Modify
| feature | P | OCCT | notes |
|---|---|---|---|
| Press/Pull (context: face → offset, edge → fillet, profile → extrude) | P1 | dispatch to Offset face / Fillet / Extrude | Fusion's signature tool; pure UX dispatch |
| Fillet (constant radius; variable P2; rule fillet P2) | P0 | `BRepFilletAPI_MakeFillet` | failure handling: report edge set, suggest smaller radius |
| Chamfer (equal, two-distance, distance-angle) | P0 | `BRepFilletAPI_MakeChamfer` | |
| Shell (remove faces, thickness in/out) | P0 | `BRepOffsetAPI_MakeThickSolid` | |
| Draft | P1 | `BRepOffsetAPI_DraftAngle` | |
| Scale (uniform, non-uniform P2) | P1 | `gp_GTrsf` | |
| Combine (join, cut, intersect; keep tools) | P0 | `BRepAlgoAPI_Fuse/Cut/Common`, `SetFuzzyValue` | |
| Offset face | P1 | `BRepOffsetAPI_MakeOffsetShape` on face set | |
| Split body / split face | P1 | `BRepAlgoAPI_Splitter` | |
| Move / Copy (free, translate, rotate, point-to-point) | P0 | `BRepBuilderAPI_Transform` | |
| Align | P2 | | |
| Delete (feature, body, face P2) | P0 | | |
| Physical material / appearance | P1 | colour, density on body; XCAF on export | |

### 5.6 Construct
Offset plane, plane at angle, midplane, tangent plane (P1), plane through 3 points, axis through
edge/two points/cylinder, point at vertex/center (P0 for plane+axis basics, P1 rest). Implemented as
`gp_Pln`/`gp_Ax1`/`gp_Pnt` features that other features can reference.

### 5.7 Inspect
Measure (distance, angle, radius, area, volume) via `BRepExtrema_DistShapeShape`, `BRepGProp`; center of
mass; bounding box; interference check (P1, `BRepAlgoAPI_Common` non-empty); curvature comb (P2).

### 5.8 Edit, timeline, browser
- Browser tree: Document → Bodies / Sketches / Construction, show/hide, rename, isolate.
- Timeline: ordered features, edit (reopen dialog), suppress, reorder (drag), roll back marker, error badge.
- Parameters dialog: named user parameters with expressions and units; every numeric input accepts an
  expression.
- Undo/redo: command-level, unlimited within session (snapshot of document JSON per command; cheap).

### 5.9 Persistent naming (the hard part, must be designed day one)

Implemented in `cadrust-core/src/naming.rs`: faces are named at birth (profile edge index, cap,
primitive side, feature-generated `f{k}`), carried by `Solid::iter_history()` face pairs through
booleans/fillet/chamfer/shell, by transformed centers through rigid moves and pattern copies, and by
nearest center as a last resort; edges are named by their face pair. Selections store name + point,
name first. Known limit: faces split by a boolean inherit one name each from the same source, so two
split halves can differ only by their `#k` edge suffixes.
Features reference topology (faces/edges) created by earlier features. After an upstream edit the OCCT
indices change. Strategy, in order:
1. Generation names: each feature records, for every face/edge it creates, a name derived from its own id
   and the sketch entity that generated it (e.g. `ex1/side[l1]`, `ex1/cap[top]`). OCCT's
   `BRepTools_History` / `BOPAlgo_History` give Generated/Modified/Deleted maps after booleans and fillets,
   which we use to carry names forward.
2. Geometric fallback: if a name is lost, match by face type + centroid + normal + area within tolerance.
3. If both fail, the feature is marked broken with the missing reference highlighted (never silently
   re-bound). This mirrors what FreeCAD 1.0 did and avoids the classic topological naming failures.

### 5.10 Tessellation and display
`BRepMesh_IncrementalMesh` with linear deflection = 0.1 % of bbox diagonal and angular 0.3 rad (quality
slider: coarse/normal/fine). Output per body: positions, normals, triangle indices, per-triangle face id,
polyline edges per edge id. Face/edge ids map to persistent names for selection. Large models: mesh per body
cached by shape hash; only changed bodies are re-sent.

### 5.11 STEP I/O: library vs own implementation
Decision: do not write a STEP geometry translator. Translating AP214/AP242 B-rep entities
(`ADVANCED_BREP_SHAPE_REPRESENTATION`, `B_SPLINE_SURFACE_WITH_KNOTS`, trimmed curves, tolerances) into a
valid B-rep is years of work and is exactly what OCCT's `STEPControl`/`STEPCAFControl` already does well.
What we do build, in Rust, is a thin Part 21 layer for the cases where OCCT is overkill or unavailable:

| need | implementation | why |
|---|---|---|
| Geometry import/export | OCCT via cadrum (`STEPCAFControl_Reader/Writer`, XCAF for names, colours, layers, assembly tree) | correct, battle-tested, AP203/214/242 |
| Header, units, product names, entity counts without loading geometry | `ruststep` Part 21 parser (pure Rust, wasm-safe) | `cadrust info` in milliseconds, works in the no-OCCT CLI build |
| Schema validation (`FILE_SCHEMA`, entity types known to AP214/242) | `espr`-generated AP214 types from ruststep | fail fast with a readable error before OCCT runs |
| Post-processing exported STEP (product name, author, organization, description, timestamp) | our own Part 21 header rewrite | OCCT writes generic headers; we fill `FILE_NAME`/`FILE_DESCRIPTION` with document metadata |
| Reverse-engineering reference | STEPcode EXPRESS schemas, OCCT's `STEPControl` source, FreeCAD import logs | when OCCT rejects a file, we diagnose with the parser and report the offending entity id |

Write options exposed in the dialog and CLI: schema (AP214 default, AP203, AP242), units (mm default),
precision mode, write names/colours on/off, one solid per body vs compound. Import options: merge solids,
sew faces (`BRepBuilderAPI_Sewing`) with tolerance, read colours/names, read layers as body groups.
Round-trip test fixtures: 10 public STEP parts (NIST MBE test cases, FreeCAD examples) checked for
face count, volume within 1e-6 relative, and name/colour preservation.

## 6. CLI and pipeline

```
cadrust run part.cadrust --set w=50 --export part.step --export part.glb
cadrust convert in.step --out out.glb --deflection 0.05
cadrust check in.step            # BRepCheck_Analyzer, reports invalid shapes, exit 1 on error
cadrust info in.step --json      # bodies, faces, bbox, volume, mass
cadrust pipeline run pipe.json   # ordered stages: import → feature ops → check → export, PDAL-style
cadrust serve --port 8080        # optional, serves the web app and a JSON-RPC endpoint
```

Pipeline JSON is the `.cadrust` timeline plus `import`/`export`/`check` stages, so a part authored in the UI
is directly runnable headless and vice versa. Exit codes and `--json` output make it CI-friendly. Logs are
English, structured (`tracing`).

## 6.5 Add-in system

Implemented (phase 5): instead of the WASM component model and WIT bindings below, v0.1 uses a
plain `wasm32-unknown-unknown` module with a four-export packed-buffer ABI (`addin_alloc`,
`addin_free`, `addin_manifest`, `addin_run`) exchanging JSON, hosted by `wasmi` natively and by the
browser's own `WebAssembly` API in the page. The request/response is the same `document.get()` +
`document.apply()` pair collapsed into one call, which keeps the sandbox tight (the add-in sees only
JSON) and needs no component toolchain. JavaScript UI add-ins follow the manifest + `run(command,
params, api)` contract. The WIT/component design stays the target for a stable v1 API.

Goal: Fusion-style add-ins (commands, custom features, importers/exporters, panels) that cannot crash or
corrupt the host and work identically in the browser and the CLI.

| layer | technology | runs in | what it can do |
|---|---|---|---|
| Core add-in | WebAssembly component with a WIT interface (`cadrust:addin`) | wasmtime on native, browser WebAssembly in the Worker | register commands, read the document, submit commands through `cadrust-cmd`, read/write its own storage, call kernel ops through the public command API only |
| UI add-in | TypeScript/ES module | browser main thread | add toolbar buttons, panels, dialogs; talks to the core through the same message protocol |
| Script | `.cadrust` pipeline JSON or a core add-in in any language that compiles to wasm32 (Rust, C/C++, AssemblyScript, Python via componentize-py) | both | parametric part generators, batch jobs |

Manifest (`addin.json`): id, version, name (localizable), entry points (`core.wasm`, `ui.js`),
permissions (document read/write, file access, network for UI add-ins only), min host version, commands
with default shortcuts and toolbar placement. Add-ins are loaded from a user folder on native and from
IndexedDB/URL in the browser; the host shows the permission list before enabling.

API surface (WIT, stable from v1): `document.get()`, `document.apply(command)`, `selection.get()`,
`events.subscribe(kind)`, `ui.notify(level, message)`, `storage.get/set`. No direct OCCT handle crosses
the boundary; add-ins work on the JSON document and command vocabulary, which keeps the sandbox tight
and the API independent of the kernel binding. Custom features implemented by an add-in are stored in
the timeline as `{"type":"AddinFeature","addin":"id","payload":...}` and re-evaluated by calling the
add-in; if the add-in is missing the feature is marked unavailable, not deleted.

Example add-ins shipped with v1: `gear-generator` (parametric spur gear), `step-batch-export`
(export every body to its own STEP), `measure-panel` (UI only). These double as the API tests.

## 7. UX spec

- Layout as Fusion: top toolbar with tabs (Solid, Surface P2, Sketch contextual), left Browser panel,
  bottom Timeline, bottom-center navigation bar (orbit/pan/zoom/fit/display/grid), right-top ViewCube,
  right-click marking menu with the last used tools, dialogs docked right-top for the active feature.
  Implemented 2026-10-18 after a pass over Autodesk's interface overview: application bar (Data Panel,
  File, Save, Undo/Redo, design tabs), workspace switcher + SOLID / UTILITIES / SKETCH tabs, groups
  CREATE / MODIFY / ASSEMBLE / CONSTRUCT / INSPECT / INSERT / SELECT / ADD-INS with drop-down command
  lists, floating Browser with visibility eyes, ViewCube with face/edge/corner picking, timeline with
  feature icons and a history marker, Change Parameters table, Data Panel drawer. Tools cadrust lacks are
  greyed out in place so the muscle memory still maps.
- Shortcuts: E extrude, Q press/pull, L line, C circle, D dimension, X construction toggle, F fillet,
  M move, S tool search palette, Ctrl+Z/Y, Esc cancel, Enter confirm, I measure (P1).
- Selection filters (bodies/faces/edges/vertices/sketch), hover highlight, window select.
- Language: English is the default and the source language of every string. Korean is the first
  additional locale. `fluent` (`.ftl`) files in `web/locales/{en,ko}`; switch in the top-right without
  reload; all strings keyed, no literals in components; missing keys fall back to English. Add-ins ship
  their own `.ftl` files.
- Themes: dark (default) and light via CSS custom properties; follows OS preference, user override stored.
- Design language: 8-px grid, 1 accent colour, icon set (Lucide, ISC licence), 13-px UI font, no gradients,
  dialogs with inline validation and expression hints.
- Accessibility: keyboard focus order in dialogs, tooltips with shortcut hints, 4.5:1 contrast in both themes.

## 8. Non-functional requirements

| area | target |
|---|---|
| Startup | WASM bundle ≤ 25 MB gzipped; first interaction < 3 s on a laptop |
| Interactive rebuild | ≤ 100 ms for ≤ 50-feature parts; fillet/boolean may stream progress |
| STEP | round-trip a 500-face part with names and colours; import 50 MB STEP in < 30 s native |
| Memory | browser tab < 1.5 GB for 1 M-triangle display |
| Determinism | same .cadrust → identical STEP on native and WASM (modulo float formatting) |
| Testing | unit tests per crate; golden STEP/glTF fixtures; property tests for sketch solver (random constraint sets stay solvable); headless Playwright smoke test for the web app |
| CI | GitHub Actions: cargo test (native), wasm-pack build, web build; OCCT prebuilt cached |
| Logging | `tracing`, English messages, feature id in every error |
| Language policy | code comments, log lines, error messages, commit messages, docs: English, one line where one line suffices; user-facing strings only through the i18n layer |

## 9. Delivery plan (2-4 h/day assumed)

| phase | weeks | deliverable |
|---|---|---|
| 0 Skeleton | 1-2 | Workspace, cadrum native + wasm build in CI, box primitive, tessellate, three.js viewer, STEP export, CLI `convert` |
| 1 Sketch | 2-3 | Sketch plane, lines/circles/arcs, ezpz constraints and dimensions, profile extraction, Extrude with New/Join/Cut |
| 2 Modify | 2 | Fillet, chamfer, shell, combine, move/copy, construction planes/axes, undo/redo, timeline edit and suppress |
| 3 Robustness | 2 | Persistent naming with history maps (done), STEP import with XCAF, ruststep header/validation path, parameters/expressions (done), error reporting (done) |
| 4 UX polish | 1-2 | Fusion layout, shortcuts, marking menu, i18n en/ko, themes, Playwright tests, docs (done) |
| 4.5 Add-ins | 1-2 | WIT interface, wasmtime + browser loader, manifest/permissions, 3 sample add-ins (done with a packed-buffer ABI under wasmi, 2 samples) |
| 5 v1 extras | ongoing | Revolve, sweep, loft, hole, patterns, mirror, press/pull, section view, draft, split |

Phase 0 is the go/no-go gate for cadrum's WASM build. If it fails, fallback A is opencascade.js in the
Worker with the Rust document model still compiled to WASM (two WASM modules, JS glue), fallback B is a
native Rust server (axum) with the browser as a thin client.

## 10. Risks

| risk | likelihood | mitigation |
|---|---|---|
| cadrum API coverage gaps (sweep, draft, history maps, XCAF) | high | fork and extend its cxx wrapper; keep an `occt-ext` crate; contribute upstream |
| OCCT WASM bundle size and compile time | medium | prebuilt OCCT artefacts cached in CI; code-split rarely used modules (IGES) |
| Topological naming breaks on edit | high | §5.9 design from day one; test suite of edit scenarios |
| Sketch solver instability (ezpz young) | medium | property tests; constraint subset for P0; planegcs fallback native |
| Fillet failures on complex edge sets (OCCT limitation) | medium | clear errors, radius suggestion, allow per-edge radius |
| WASM component model support in browsers needs a JS shim (jco) | medium | use jco transpile at build time; fall back to plain wasm32 exports with a JSON ABI |
| Scope creep toward "all of Fusion" | high | P0/P1/P2 table is the contract; v1 = P0 + P1 only |
| Licence compliance for static OCCT in WASM | low | documented attribution, unmodified OCCT, relink info, legal note in README |

## 11. Open questions
1. Verify licences from the LICENSE files: cadrum, kcl-ezpz. Both reported MIT/Apache; confirm before coding.
2. React vs Svelte for `web/`: React chosen for hiring familiarity; Svelte is lighter. Decide at phase 0.
3. Should `cadrust-server` exist in v1 at all, or only `cadrust serve` static hosting? Default: static only.
4. Units: mm default; inch support via parameter units only in v1.
5. ruststep maturity for AP242 schema generation: if `espr` cannot compile AP242, keep AP214 types only and treat AP242 files as opaque for validation.

## 12. Glossary
B-rep (boundary representation: solids as faces/edges/vertices with geometry), OCCT (Open CASCADE
Technology, the open-source B-rep kernel), STEP AP214/AP242 (ISO 10303 exchange formats; 242 adds PMI),
XCAF (OCCT's document framework carrying names, colours, assembly structure), persistent naming (stable
identifiers for topology across parametric rebuilds), tessellation (triangulating B-rep faces for display),
DOF (degrees of freedom left in a sketch), fuzzy boolean (tolerance-relaxed boolean to survive near-coincident
geometry), MVC (Model = document and kernel, View = web UI, Controller = command bus / CLI).

## 13. References
- cadrum crate: https://docs.rs/cadrum (OCCT 8.0 static, native + wasm32)
- opencascade-rs: https://github.com/bschwind/opencascade-rs, fork https://github.com/mkovaxx/opencascade-rs
- ezpz solver: https://docs.rs/ezpz, announcement https://zoo.dev/blog/announcing-solver
- OCCT licence and exception: https://dev.opencascade.org/doc/overview/html/occt_public_license.html, https://spdx.org/licenses/OCCT-exception-1.0.html
- CADmium: https://github.com/CADmium-Co/CADmium (Elastic 2.0)
- Chili3D: https://github.com/xiangechen/chili3d (AGPL-3.0)
- opencascade.js: https://github.com/donalffons/opencascade.js (LGPL-2.1)
- truck: https://github.com/ricosjp/truck (Apache-2.0)
- ruststep / espr: https://github.com/ricosjp/ruststep (MIT/Apache-2.0)
- STEPcode: https://github.com/stepcode/stepcode (BSD-3)
- NIST MBE PMI STEP test models: https://www.nist.gov/ctl/smart-connected-systems-division/smart-connected-manufacturing-systems-group/mbe-pmi-0
- WebAssembly Component Model / WIT: https://component-model.bytecodealliance.org/ ; jco: https://github.com/bytecodealliance/jco ; wasmtime: https://wasmtime.dev
- Fusion 360 add-in API (reference for scope): https://help.autodesk.com/view/fusion360/ENU/?guid=GUID-A92A4B10-3781-4925-94C6-47DA85A4F65A
- CadQuery feature → OCCT mapping reference: https://github.com/CadQuery/cadquery
- FreeCAD topological naming (1.0 release notes): https://wiki.freecad.org/Release_notes_1.0
- Fusion 360 keyboard shortcuts: https://www.autodesk.com/support/technical/article/caas/sfdcarticles/sfdcarticles/Fusion-360-Keyboard-Shortcuts.html
- OCCT user guides (modeling algorithms, STEP, XDE): https://dev.opencascade.org/doc/overview/html/

# MuseScore Studio: Repository Structure

Reference notes for navigating this codebase. Generated from a read-only survey of the
working tree at commit `2243418b29` (branch `main`, fork `github.com/xvb8/MuseScore`).

Version per `version.cmake`: **MuseScore Studio 5.0.0**, with `MUSE_APP_UNSTABLE ON`
and `MUSE_APP_IS_PRERELEASE ON` — this is the in-development 5.x line. (For contrast,
`build_nightly.yml` tracks `CURRENT_RELEASE_BRANCH: 4.7`.)

## Top-level layout

| Path | What it is |
| --- | --- |
| `src/` | All MuseScore-specific application code. 19 modules. |
| `muse/` | **MuseFramework** git submodule (`musescore/muse_framework`). The generic, app-agnostic platform layer. |
| `muse_deps/` | Third-party dependency recipes and prebuilt mirror, git submodule (`musescore/muse_deps`). |
| `dev-docs/` | Developer documentation, git submodule (`musescore/dev-docs`). |
| `buildscripts/` | `cmake/` modules, `ci/` per-platform scripts, `packaging/`, `tools/`. |
| `share/` | Installed runtime resources: instruments, templates, styles, locales, plugins, soundfonts, workspaces, manual. Has its own `CMakeLists.txt`. |
| `fonts/` | Music and text fonts (Leland, Bravura, Edwin, Petaluma, MuseJazz, Gootville, Campania, Finale*, SMuFL metadata) plus FontForge `.sfd` sources. |
| `test/` | Shared score-file **fixtures** (`.mscz`, `.mscx`, `.mid`, MusicXML) — not a test harness. Real unit tests live per-module under `src/*/tests/`. |
| `vtest/` | Visual regression tests: compares current draw data against `reference/`. Built under `MUSESCORE_BUILD_CONFIGURATION=vtest`. |
| `tools/` | Maintainer utilities: `codestyle`, `fonttools`, `miditools`, `translations`, `jsdoc`, `bww2mxml`, `check_build_without_qt`, `harmony_to_diagram`, `soundfonts`, `release_notes`, `ziprw`, `checkheaders`. |
| `demos/` | Bundled demo scores, installed with the app. |
| `sandbox/` | Standalone experimental mini-apps with their own `CMakeLists.txt`, outside the main build: `cpad/`, `engraving/`, `web_sandbox/`. |
| `hooks/` | `install.sh` / `uninstall.sh` / `pre-commit` — uncrustify auto-formatter. Honors per-directory `.untidy` opt-out markers. |
| `docs/` | Plugin/JS API documentation via jsdoc. `apidocs_static/`, `old_docs/`, and a `README.md` describing the generation pipeline. |
| `.github/` | 22 workflows plus issue/PR templates and compiler problem matchers. |
| `.tx/` | Transifex config. Two QT-type resources: `musescore` and `instruments`, both from `share/locale/`. |
| `wf` | **A bash script**, not a directory. Manages the `muse` submodule when developing framework changes alongside app changes: `use-fork`, `use-origin`, `make-branch`, `sync-origin`. |
| `Compatibility` | **A plain text file.** Historical notes on MuseScore 1.2 score compatibility (beams spanning measures). |
| `mscore3.txt` | Legacy MuseScore 3 design document: the layout redesign, `Shape`-based collision outlines, incremental range layout, automatic vertical staff spacing. Historical rationale. |

Also at root: `.clangd`, `.coderabbit.yaml` (AI review bot), `.git-blame-ignore-revs` (mass-reformat commits to skip in blame), `.vscode_template/`, `Doxyfile.plugins`, `LICENSE.txt`.

### Submodules

All three are initialized and at their pinned commits:

```
git submodule status
 5065b608d90cbecea7234377c5c7c75c6cf9d0b0 dev-docs (5065b60)
 585221a5d342fa6ab52154378ad9d8a896a3b9a2 muse (585221a5d)
 4dc7c4db3a6156cbf09738815b5768e77f0cc603 muse_deps (deps-20261112-130945-1-g4dc7c4d)
```

If a fresh clone omits `--recurse-submodules`, fix it with:

```bash
git submodule update --init --recursive
```

Without them the build fails early — `buildscripts/cmake/SetupDependencies.cmake`
hard-errors on a missing `muse_deps`, and the root `CMakeLists.txt` cannot
`add_subdirectory(muse/framework)` or find ~11 CMake modules that live there
(`GetPaths`, `GetBuildType`, `GetPlatformInfo`, `SetupBuildEnvironment`, `SetupQt6`,
`SetupCompilerCache`, `CheckCompilerVersion`, `TryUseLocalRevision`,
`MuseDeclareOptions`, `MuseSetupConfiguration`, `ExtDepsManifest`). The codestyle CI
checker also lives in the submodule
(`muse/buildscripts/ci/checkcodestyle/checkcodestyle.cmake`).

## MuseFramework (`muse/`) and its relation to `src/`

MuseFramework is the generic, app-agnostic foundation; `src/` is the notation
application built on top of it. The framework tree is compiled as part of the *same*
CMake project, in this order (root `CMakeLists.txt`):

```
add_subdirectory(share)
add_subdirectory(${MUSE_FRAMEWORK_SRC_PATH})   # muse/framework
add_subdirectory(src)
```

`muse/` top level: `framework/`, `buildscripts/`, `tools/`, `docs/`, `.github/`.

**Naming convention split — worth memorizing:**

| Prefix | Owner |
| --- | --- |
| `MUSE_*`, `MUSE_MODULE_*` | MuseFramework options |
| `MUE_*`, `MUSESCORE_*` | MuseScore Studio app options |

The app declares its toggles, then `SetupConfigure.cmake` calls the framework's
`MuseSetupConfiguration.cmake` to consume them.

### Framework modules (`muse/framework/`, 35 directories)

```
accessibility  actions     audio        audioplugins  automation
cloud          diagnostics dockwindow   draw          extensions
global         interactive languages    learn         media
midi           midiremote  mpe          multiwindows  musesampler
network        rcommand    rcontrol     shortcuts     shortcuts_v2
stubs          testflow    testing      toast         tours
ui             uicomponents update      vst           workspace
```

Plus `cmake/`, which is build infrastructure rather than a module. Note both
`shortcuts` and `shortcuts_v2` exist — the app selects v2 via
`MUSE_MODULE_SHORTCUTS_V2 ON`.

`framework/stubs/` mirrors the app's own `src/stubs/`: nearly every optional framework
module has a paired stub (`framework/audio/main/audiomodule.h` vs
`framework/stubs/audio/audiostubmodule.h`), selected by `#ifdef MUSE_MODULE_*` in
`src/app/appfactory.cpp`. This is how the `app-web` and `vtest` configurations compile
with most modules switched off.

Most relevant to score playback work: `audio` (engine, drivers, worker), `mpe`
(Musical Performance Expression — the event/articulation model engraving targets),
`musesampler`, `vst`, `audioplugins`, `midi`.

**`framework/automation/` is currently an empty placeholder** — just
`automationmodule.cpp` (which only returns its name), a `CMakeLists.txt`, and an empty
`Muse.Automation` QML module. The actual automation primitives live in `mpe`; see below.

`muse_deps/` holds 57 dependency recipes (`recipes/<name>/spec.cmake` with
`DEP_VERSION`, `DEP_SOURCE_URL`, `DEP_SOURCE_SHA256`), `buildtools/manifest.cmake`, and
a `prebuilt.lock` + `prebuilt_url.txt` pair so CI can fetch prebuilt binaries instead
of compiling from source. Beyond the audio codecs the app requires, it carries
`uncrustify` (for the formatting hook), `googletest`, `vst3sdk`, `lv2sdk`, `asiosdk`,
`crashpad_client`, and `kddockwidgets`.

## Code size by module

Counted across `.cpp`, `.h`, `.qml` under `src/`:

| Module | Files | ~KLOC |
| --- | --- | --- |
| `engraving` | 1048 | 345 |
| `importexport` | 424 | 142 |
| `notationscene` | 287 | 46 |
| `project` | 255 | 39 |
| `propertiespanel` | 352 | 35 |
| `notation` | 129 | 31 |
| `palette` | 76 | 16 |
| `playback` | 88 | 14 |
| `instrumentsscene` | 70 | 12 |
| `appshell` | 98 | 12 |
| `preferences` | 103 | 9 |
| `braille` | 29 | 9 |
| `stubs` | 75 | 5 |
| `musesounds` | 39 | 4 |
| `converter` | 23 | 3 |
| `web` | 25 | 3 |
| `app` | 11 | 2 |
| `print` | 5 | 0.2 |
| `macos_integration` | 1 | 0.1 |

`engraving` is over half the codebase. Everything else is comparatively thin.

## The module system

MuseScore is built as a set of registered modules wired by dependency injection, not
by direct linking between features.

### Conventions

Each module under `src/` follows the same shape:

- `<name>module.cpp` / `.h` — the module class. Lifecycle hooks: `moduleName()`,
  `registerExports()`, `resolveImports()`, `registerApi()`, `registerResources()`,
  `registerUiTypes()`, `onInit(RunMode)`, `onAllInited()`, `onDeinit()`.
- `i*.h` at module root — **public interfaces**, the only thing other modules should
  include (e.g. `src/notation/inotation.h`, `inotationautomation.h`,
  `src/project/inotationproject.h`). Internal-only interfaces live *inside* `internal/`
  as `i*.h` (`src/project/internal/iprojectmigrator.h`,
  `src/notation/internal/igetscore.h`). Forward decls use `_fwd.h`.
- `internal/` — concrete implementations, configuration (`<name>configuration.cpp`),
  action/command controllers, and `internal/platform/<macos|windows>/` OS-specific code.
- `types/` — plain data/enum headers (`src/notation/types/automationlane.h`).
- `qml/MuseScore/<Module>/` — QML UI. Note: **the C++ QML-facing view models live next
  to their `.qml` files here**, not in a `view/` directory
  (`playbacktoolbarmodel.cpp` sits beside `PlaybackToolBar.qml`).
- `tests/` with `tests/mocks/`, `utils/`, `data/` — GoogleTest + gmock. Opt-in via
  `MUE_BUILD_<NAME>_TESTS`. 24 test directories exist.
- `widgets/` — leftover Qt Widgets code (`.ui` files) where QML hasn't taken over yet.
- `CMakeLists.txt` — per-module target, gated on `MUE_BUILD_<MODULE>_MODULE`. Large
  modules split source lists into `.cmake` fragments (`dom/dom.cmake`,
  `rendering/score/rendering.cmake`, `playback/playback.cmake`).

Two things that look like conventions but aren't:

- **`view/` is vestigial.** Only two stray files remain
  (`src/instrumentsscene/view/staffcontroltreeitem.h`, a duplicate `.ui` in
  `src/palette/view/widgets/`). Don't create new ones.
- **`dev/` does not exist.** Dev tooling is `devtools/` (`src/engraving/devtools/`) or
  a `DevTools/` QML directory.

### Dependency injection is two-level

```cpp
// Global singletons
globalIoc()->registerExport<INotationConfiguration>(moduleName(), m_configuration);
auto writers = globalIoc()->resolve<project::INotationWritersRegister>(mname);
```

Per-window / per-document services use a **context** object returned from
`newContext(ContextPtr)` — `EngravingContext`, `NotationContext`, `PlaybackContext`,
`ProjectContext`, `AppShellContext` — each with its own `registerExports()` /
`resolveImports()` / `onInit()` / `onDeinit()`. This is the newer pattern and is
spreading through the codebase; prefer it for anything document-scoped.

### Commands subsystem

Newer modules carry a uniform quadruple registered against
`muse::rcommand::ICommandsRegister` / `ICommandsState`:

```
<name>commands.h
internal/<name>commandsregister.cpp
internal/<name>commandsstate.cpp
internal/<name>commandscontroller.cpp
```

Present in `appshell`, `notation`, `playback`, `project`, `palette`,
`instrumentsscene`.

### Dialog registration

`resolveImports()` registers QML dialogs and pages with `IInteractiveUriRegister`,
mapping a `musescore://...` URI to `("MuseScore.<Module>", "<DialogName>")`. `project`
alone registers 13 such URIs.

### Resources

Three mechanisms coexist — match whichever the module already uses:

- Classic `.qrc` (`engraving/engraving.qrc`, `app/app.qrc`) with `Q_INIT_RESOURCE` in
  `registerResources()`.
- `muse_module_add_qrc(<target> ...)`, with a `BIG_RESOURCES` variant for fonts.
- Target-based `qt_add_resources(<target> <name> PREFIX / FILES ...)` (appshell).

QML modules are created with `muse_create_qml_module(<name>_qml FOR <name>)` followed
by `qt_add_qml_module(... URI MuseScore.<Module> ...)`, gated on
`MUE_BUILD_<NAME>_QML`.

### Composition root

`src/app/appfactory.cpp` is the single place where the whole application is assembled.
It builds either a GUI app or a console app and calls `app->addModule(...)` for every
framework and MuseScore module, each wrapped in an `#ifdef`:

```cpp
app->addModule(new muse::audio::AudioModule());
app->addModule(new muse::automation::AutomationModule());
app->addModule(new mu::engraving::EngravingModule());
app->addModule(new mu::notation::NotationModule());
app->addModule(new mu::playback::PlaybackModule());
```

Two notes worth remembering:

- **Diagnostics must be first** — it installs the crash handler. Order is otherwise
  irrelevant, which is the point of the DI design.
- **`src/stubs/`** provides no-op implementations of module interfaces so a module can
  be compiled out (`MUE_BUILD_NOTATION_MODULE=OFF`) while dependents still link. Same
  pattern exists in the framework (`framework/stubs/...`). This is how the headless
  converter and WASM builds stay small.

## `src/engraving` — the score model and layout engine

The core library (historically `libmscore`). Always built, never optional.

| Path | Role |
| --- | --- |
| `dom/` | **362 files.** The score object model. One file per element type. |
| `editing/` | ~70 undoable edit operations, one unit per feature. `editing/transaction/` holds `undostack`, `transaction`, `undoablecommand`. |
| `rendering/` | Layout. `score/` (~50 pairs — the full layout engine), `single/` (isolated element, for palette previews), `editmode/`. |
| `rw/` | Read/write. Versioned readers `read114` → `read500`, plus `write/` and `compat/`. Has its own `README.md`. |
| `automation/` | Automation curve data model (see below). |
| `playback/` | Score → MPE event generation: `metaparsers/`, `renderers/`, `mapping/`, `filters/`, `utils/`. |
| `style/` | `style`, `styledef`, `textstyle`, `pagestyle`, `defaultstyle`. |
| `types/` | `types.h`, `fraction.h`, `spatium.h`, `propertyvalue`, `symid.h`/`symnames`, `constants.h`. |
| `infrastructure/` | `mscreader`/`mscwriter` (MSCZ container I/O), `smufl`, `shape`/`skyline` (collision geometry), `eid`/`eidregister` (stable element IDs), `rtti.h`, `ld_access.h`. |
| `compat/` | Legacy-format shims. `compat/midi/` is the legacy MIDI renderer (`pitchwheelrenderer`, `velocitymap`, `pausemap`, `easeinout`). |
| `api/v1/` | QML plugin API v1, with its own test target. |
| `devtools/` | `engravingelementsprovider`; `devtools/drawdata/` is the draw-data regression harness that `vtest/` drives. |
| `data/` | Shipped data: 13 `fonts_*.qrc` bundles, legacy `.mss` styles, chord description XMLs, `harmony_to_diagram.xml`. |
| `accessibility/`, `qml/`, `tests/`, `thirdparty/` | Supporting. `thirdparty/` has `dtl` (diff template library) and `intervaltree`. |

`tests/` holds ~85 `*_tests.cpp` plus ~70 `*_data/` fixture directories.

### Layout engine detail

`rendering/score/` is where spacing and positioning happen — directly relevant to any
proportional-spacing work: `horizontalspacing`, `scorehorizontalviewlayout`,
`scoreverticalviewlayout`, `measurelayout`, `segmentlayout`, `systemlayout`,
`pagelayout`, `beamlayout`, `chordlayout`, `dynamicslayout`, `tupletlayout`,
`autoplace`, plus the big per-type dispatchers `tlayout`/`tdraw` and pass
infrastructure (`passbase`, `passlayoutindependentitems`, `passresetlayoutdata`).

### Versioned readers

`rw/read114/`, `read206/`, `read302/`, `read400/`, `read410/`, `read460/`, `read500/`.
Per that directory's `README.md`: 114–400 share code, `read410` is a fork of `read400`,
and `read460` is a fork of `read410`. Adding a new score-file field means touching
`write/` and the newest reader.

### `dom/` highlights

Largest files: `score.cpp` (168 KB), `note.cpp` (146 KB), `measure.cpp` (115 KB),
`textbase.cpp` (104 KB), `segment.cpp` (97 KB), `engravingitem.cpp` (94 KB),
`property.cpp` (92 KB), `chord.cpp` (84 KB).

Hierarchy: `EngravingObject` → `EngravingItem` → everything else. `Score` /
`MasterScore` own the document; `Measure` → `Segment` → `ChordRest` → `Chord`/`Rest` →
`Note` is the vertical spine. `Spanner` covers anything spanning time (`Hairpin`,
`Slur`, `Ottava`, `Pedal`, `Trill`, `Volta`).

Directly relevant to notation-driven expression: `dynamic.h`, `hairpin.h`,
`dynamichairpingroup.h`, `expression.h`, `articulation.h`, `ornament.h`,
`playtechannotation.h`, `soundflag.h`, `gradualtempochange.h`, `tempotext.h`,
`tempotimeline.h`, `instrument.h`, `stringdata.h`.

`engravingmodule.cpp` `onInit()` registers ~40 font faces into
`muse::draw::IFontsDatabase`, calls `Smufl::init()`, loads `DefaultStyle`, and
initializes `Drumset` / `FiguredBass` / `StaffType`.

Build carve-outs to be aware of: `ENGRAVING_NO_ACCESSIBILITY`, `ENGRAVING_NO_INTERNAL`,
`ENGRAVING_NO_API`, `MUE_ENABLE_ENGRAVING_{RENDER_DEBUG,LD_ACCESS,LD_PASSES}`.

## `src/engraving/automation` — existing CC-style automation

This is new, actively developed, and directly overlaps the project in
`agentdocs/writeup.md`. Worth reading before designing anything in that space.

```
src/engraving/automation/
├── automationtypes.h            # core types
├── automationtypes_fwd.h
├── automationdata.cpp/.h
├── dynamicvalues.h
├── tempovalues.h
└── internal/
    ├── automationrw.cpp/.h              # JSON persistence
    └── scoreautomationcontroller.cpp/.h # generates curves from score items
```

Key design points from `automationtypes.h`:

- `AutomationPoint` wraps `muse::mpe::AutomationPoint` and carries
  `std::optional<EID> itemId` — *"valid if it was created from an engraving item
  (e.g., Dynamic)"* — plus `bool generated` (true until the user edits it). So
  score-derived and hand-drawn points coexist in one curve, and overrides are tracked.
- `AutomationCurve = SharedMap<utick_t, AutomationPoint>` — keyed by **musical tick**,
  not absolute time.
- Points support `Ease`, `ArrivalFromPrevious`, `ExplicitArrival`, `InValue`, and
  `evaluateCurveAt()` interpolates between them.
- `AutomationType`: `Dynamics`, `Tempo`, `Volume`, `Pan`.
- `AutomationCurveKey` scopes a curve via `std::variant<std::monostate, Instrument,
  StaffVoice>` — whole score (tempo), per instrument track (volume/pan), or per
  staff-and-optional-voice (dynamics). `withoutVoice()` gets the shared staff curve.
- Edits are a command list: `AutomationPointEdit` with `SetPoint` / `MovePoint` /
  `ErasePoint`, and `AutomationChanges` reports affected keys and a tick range for
  incremental invalidation.

The UI side:

- `src/notation/inotationautomation.h` — `isAutomationModeEnabled()`,
  `automationData()`, `editPoints(key, edits)`.
- `src/notation/types/automationlane.h` — lanes: `Dynamics`, `Tempo`, `Volume`, `Pan`,
  `MasterVolume`, `MasterPan`.
- `src/notationscene/qml/MuseScore/NotationScene/notationautomationcontroller.cpp` —
  QML-facing controller.
- `src/engraving/editing/editautomationpoints.cpp` — undoable edit commands.
- Tests: `src/engraving/tests/automation/` (`automation_tests.cpp`,
  `automationrw_tests.cpp`, `scoreautomationcontroller_tests.cpp`).

### The framework primitives underneath: `muse/framework/mpe/automationpoint.h`

The engraving types wrap `muse::mpe::AutomationPoint`, which is where the curve math
actually lives. Note this is in **`mpe`**, not `framework/automation/` (that module is
an empty placeholder).

The value model is a two-sided point, which is more expressive than a single value:

```cpp
struct AutomationPoint {
    struct Ease { real_t t = 0.5; real_t value = 0.5; };   // both [0, 1]
    struct ArrivalFromPrevious {};                          // inherit preceding value
    struct ExplicitArrival { real_t value; Ease ease; };    // step to a new value
    using InValue = std::variant<ArrivalFromPrevious, ExplicitArrival>;

    InValue inValue = ArrivalFromPrevious {};
    real_t outValue = 0.;
};
```

`inValue` / `outValue` means a single point can be **discontinuous** — arrive at one
value and leave at another. That is exactly the representation needed for *subito
piano*, *fp*, and sforzando: a step rather than a ramp. The point-event problem from
`writeup.md` is already solved at the data level.

Curve shaping is also handled. `evaluateAt()` splits each segment into **two quadratic
Bezier arcs meeting at the bend point**, sharing a tangent there, so `Ease{t, value}`
gives a draggable handle where `t` is the bend's horizontal position and `value` its
vertical fraction. With no ease it degenerates to a straight line. That covers
ease-in / ease-out / exponential-ish shapes from one control.

Other useful functions:

- `resolveInValue(point, prevOutValue)` — resolves what a point's arrival value
  actually is.
- `evaluateCurveAt(curve, pos)` — interpolates anywhere on the curve; holds the first
  point's value backwards in time, "matching standard envelope semantics."
- `resampleCurve(curve, stepInterval, onSample)` — discretizes "for consumers that
  can't evaluate on read," skipping flat segments. This is the bridge for anything
  needing actual CC messages at a fixed resolution.

Values are normalized to `[0, 1]` and clamped, not 0–127. Mapping to MIDI CC range is
a separate concern.

**Takeaway:** the generic curve engine (two-sided points, Bezier easing, evaluation,
resampling) is framework-level and reusable; `src/engraving/automation/` adds the
score-specific parts (tick anchoring, `EID` back-references, scoping by
instrument/staff/voice, undo). A notation-first feature would extend the engraving
layer and the `notationscene` UI, not rewrite the math.

Recent commit history in this area shows direction of travel:

```
55609857d3 Fix dead zones in Dynamics automation lane
cda41540b4 Automation - escape to cancel current edit (bump framework)
eabd76b78b Add ability to override TempoTimeline
661d88cbab Replace legacy TempoMap with TempoTimeline
a00c7467ca Rename Bend -> Ease
f4173b060f Evaluate dynamics curves in the audio engine instead of pre-baking a fixed-resolution map
```

Several open questions in `writeup.md` already have answers in-tree: curves are
tick-anchored, dynamics curves are evaluated live in the audio engine rather than
pre-baked, hairpin/dynamic-derived points keep a back-reference to their engraving
item, and a tempo lane exists.

## Other `src/` modules

**`app`** — the executable target (`MuseScoreStudio`, output `mscore`, or
`MuseScore5.exe` on Windows). `main.cpp`, `appfactory.cpp` (composition root),
`internal/guiapp.cpp`, `internal/consoleapp.cpp` + `commandlineparser.cpp`,
`webapi_export.cpp` for WASM. `configs/` holds runtime `.cfg` defaults (shortcuts,
themes, languages, workspaces). Its `CMakeLists.txt` has an `add_to_link_if_exists()`
macro handling the three-state module situation: enabled / stubbed / absent.

**`appshell`** — main window, menus/toolbars, page layout, startup sequencing, session
restore, splash screen. `internal/startupscenario.cpp`, `internal/sessionsmanager.cpp`.
QML has `HomePage/`, `NotationPage/`, `PublishPage/`, `FirstLaunchSetup/` wizard, and a
large `DevTools/` tree. Platform menu bars are selected per-OS in CMake
(`platform/mac|win|linux/Main.qml`).

**`notation`** — the model-side service façade over `engraving`. Deliberately
**headless, no QML**. ~24 public interfaces: `inotation.h`, `inotationinteraction.h`,
`inotationnoteinput.h`, `inotationparts.h`, `inotationplayback.h`,
`inotationundostack.h`, `inotationselection.h`, `inotationautomation.h`,
`imasternotation.h`, `iexcerptnotation.h`, `inotationviewstate.h`, and more.
`internal/notationinteraction.cpp` is the largest file. Owns `InstrumentsRepository`
(loads `instruments.xml` / `orders.xml`) and registers the `mscz`/`mscx`/`spos`/`mpos`
writers into `project`'s registry. `notationplayback.cpp` is swapped for
`notationplaybackstub.cpp` when `MUE_BUILD_ENGRAVING_PLAYBACK` is off.

**`notationscene`** — the notation **UI**: score canvas and everything around it.
`NotationView.qml` + `notationpaintview`, `abstractnotationpaintview`,
`notationviewinputcontroller`, `continuouspanel`, `playbackcursor`, `noteinputcursor`,
`loopmarker`, `notationruler`, `notationnavigator`, `notationautomationcontroller`.
Also `elementpopups/` (per-element: Dynamic, Capo, HarpPedal, StringTunings, …),
`styledialog/` (~25 style pages), `percussionpanel/`, `pianokeyboard/`,
`selectionfilter/`, and a `widgets/` tree of legacy Qt Designer dialogs
(`editstyle`, `editstaff`, `measureproperties`, `timeline`, …). **This is where a
pageless, lane-based view would be built.**

**`playback`** — playback *orchestration and UI*: transport, mixer, sound profiles,
online/cloud sounds, drumset loading. `internal/playbackcontroller.cpp` is the main
controller. `internal/onlinesoundscontroller.cpp` handles cloud rendering;
`internal/knownaudiopluginsconfigurator.cpp` must run before the audio-plugin cache
loads. QML: `MixerPanel.qml` + `mixerchannelitem`, resource items
(`inputresourceitem`, `outputresourceitem`, `auxsenditem`), `SoundFlagPopup.qml`.

Note the three-way split: event *generation* is `src/engraving/playback/`, audio
*rendering* is framework `muse/framework/audio` + `mpe`, and this module is the UI and
control layer.

**`project`** — document lifecycle. Owns `NotationProject` (master notation + metadata
+ audio settings), the reader/writer registries that `importexport` plugs into, and all
user scenarios: open, save, save-to-cloud, close, export, migrate, autosave, recent
files, templates, cloud-score deletion. Registers 13 `musescore://project/...` URIs and
has 11 test suites. `internal/platform/{macos,windows}/` integrate OS recent-documents
lists. `internal/mscmetareader.cpp` reads metadata without a full load.

**`importexport`** — 17 independent sibling modules (CMake targets `iex_*`), each
registering readers/writers into `project`. `musicxml` has a two-pass importer;
`guitarpro` is the largest (GP4/5/6/7 + Power Tab, with a GP6/7 DOM model and a
`guitarbendimport/` subsystem); `midi` has ~25 `importmidi_*` units for quantization,
tuplet detection, and voice separation. Also `mei`, `mnx`, `capella`, `bb`, `bww`,
`musedata`, `ove`, `tabledit`, `mlscore` (**off by default**), `audioexport`
(wav/mp3/ogg/flac/aac), `imagesexport` (png/svg/pdf), `videoexport`, `lyricsexport`.

**`propertiespanel`** — the Properties panel (successor to the Inspector). Mostly QML,
with `notation/` containing ~37 per-element-type subdirectories and `types/` holding 21
QML-facing enum headers. `internal/elementrepositoryservice.cpp` tracks and filters the
selection.

**`palette`** — palettes panel: tree model, cells, drag-and-drop application to the
score, plus master palette / symbol / time-sig / key-sig / note-groups editors. Legacy
`.ui` dialogs survive in `widgets/`.

**`instrumentsscene`** — instruments/layout panel and the choose-instruments flow.

**`preferences`** — the Preferences dialog. Essentially **pure QML**; the C++ side is
only module registration. One page + model per section.

**`braille`** — braille output and braille note input, built on liblouis. `tables/`
holds the translation tables (`en-us-symbols.mus`, `smufl-symbols.mus`, …).

**`converter`** — headless batch conversion driver used by the console app; also
exposed as `MuseApi.Converter` via `registerApi()`.

**`musesounds`** — MuseSounds library browsing, update checks, MuseSampler status.

**`print`** — smallest real module: a `QPrinter`-based print provider. 5 files.

**`web`** — the WASM configuration (`MUE_CONFIGURATION_IS_APPWEB`). `web/appshell/` is
a slimmed-down *replacement* for `src/appshell`; `web/appjs/` is the JS bridge
(`muapi.js`, `audioworker.js`, `audio_worklet_processor.js`); `web/audioengine/` is the
WebAudio engine.

**`macos_integration`** — macOS-only Quick Look thumbnail/preview provider, part Swift.
The **only** module with no `*module.cpp` — it is not a Muse framework module.

**`stubs`** — no-op implementations so the app links unconditionally when a module is
compiled out. Pattern: `<name>stubmodule.cpp` + `<iface>stub.cpp`, often with an empty
`qml/` dir to satisfy QML import targets. Covers `braille`, `instrumentsscene`,
`musesounds`, `notation`, `notationscene`, `palette`, `playback`, `project`,
`propertiespanel`, and `importexport/{mei,midi,mnx,musicxml,ove}`.

Modules with **no** tests: `app`, `appshell`, `macos_integration`, `palette`,
`preferences`, `print`, `propertiespanel`, `stubs`, `web`.

## Build system

| File | Purpose |
| --- | --- |
| `CMakeLists.txt` | Root. C++20, declares all `MUE_BUILD_*` options, orders the subdirectories. |
| `SetupConfigure.cmake` | 409 lines. Translates two coarse knobs into hundreds of concrete module toggles. |
| `build.cmake` | Script-mode driver — the documented way to build (`cmake -P build.cmake`). |
| `version.cmake` | App name, version 5.0.0, release channel from `MUSE_APP_BUILD_MODE`. |
| `CMakePresets.json` | 11 presets (schema v6). |
| `ninja_build.sh` | The primary Linux/macOS/CI build script, driven entirely by env vars. |
| `ninja_build.bat` | Windows wrapper: `vswhere` → `vcvars64.bat` → `bash ./ninja_build.sh`. |
| `pyproject.toml` / `pdm.lock` | PDM dev venv for the Python scripts in `buildscripts/`. Python ≥ 3.10. |
| `Doxyfile.plugins` | Plugin API docs. |

### Two knobs drive everything

```
MUSESCORE_BUILD_CONFIGURATION = app | app-portable | app-web | vtest | utest   (default: app)
MUSE_APP_BUILD_MODE           = dev | testing | release                        (default: dev)
```

`SetupConfigure.cmake` expands these. Highlights:

- **app / app-portable** — `MUSE_MODULE_AUDIO_WORKMODE 3` (driver callback + worker
  RPC). Portable sets `WIN_PORTABLE ON` and disables the updater.
- **app-web** — large opt-out list. WASM uses workmode 2 and
  `MUSE_THREADS_SUPPORT OFF` ("engine runs in the audio worklet, a single thread").
  Disables linguist tools, QProcess, most impex modules, splash screen, console app.
- **vtest** — enables ASan, keeps only `converter` + `notation` + `project` +
  `imagesexport` + `guitarpro`; `MUSE_MODULE_UI ON` but `UI_QML OFF`.
- **utest** — `MUSE_ENABLE_UNIT_TESTS ON`, ASan on unless coverage is on. Under
  coverage it deliberately builds *every* module "so that the ones without tests are
  counted as uncovered."
- **Dependency cascades** — e.g. `NOT MUE_BUILD_NOTATION_MODULE` turns off
  notationscene, project, and palette; `NOT MUSE_MODULE_UI_QML` turns off every
  `*_QML`; ASan forces `MUSE_ENABLE_CUSTOM_ALLOCATOR OFF`.

A gitignored `SetupConfigure.local.cmake` can replace `SetupConfigure.cmake` entirely,
and `build.cmake` honors a gitignored `build_overrides.cmake`.

### Toolchain requirements

CMake **3.28** minimum (3.31 on Apple). **C++20**. Minimum compilers: Clang **20**,
AppleClang **21**, GCC **14**, MSVC **19.40**.

### Building

```bash
cmake -P build.cmake -DCMAKE_BUILD_TYPE=Release        # build
cmake -P build.cmake -DCMAKE_BUILD_TYPE=Release run    # run
cmake -P build.cmake -DCMAKE_BUILD_TYPE=Release clean  # wipe build dir
```

Default build type if omitted is `RelWithDebInfo`. On macOS add `-G Ninja` (required
for the Swift components). `build.cmake` outputs to
`builds/<Platform>-Qt<ver>-<compiler>-<gen>-<type>`, abbreviated on Windows to stay
under the 260-character path limit, and requires `qmake6`/`qmake` on `PATH` (or
`$QTDIR`).

`ninja_build.sh` is the alternative, used by CI. Targets: `release`, `debug`,
`relwithdebinfo`, `install*`, `clean`, `compile_commands`, `revision`, `appimage*`.
Everything is an overridable `MUSESCORE_*` env var.

Presets: `x64-Debug/Release/RelWithDebInfo/MinSizeRel` (VS generator),
`Qt-Clang-Debug/Release`, `Qt-Msvc-Debug` (Ninja, needs `$QT_DIR`), and
`compile_commands` — which the root `CMakeLists.txt` invokes recursively when
`MUSE_GENERATE_COMPILE_COMMANDS=ON`, then copies the result to the source root.

### Notable options

Every feature module has `MUE_BUILD_<NAME>_MODULE`, usually with matching `_QML` and
`_TESTS` options defaulting to the module's own value. All default ON except
`MUE_BUILD_IMPEXP_MLSCORE_MODULE`. Engraving splits further: `MUE_BUILD_ENGRAVING_QML`,
`_TESTS`, `_DEVTOOLS`, `_PLAYBACK`.

Framework-side: `MUSE_COMPILE_USE_UNITY` (ON), `MUSE_COMPILE_USE_COMPILER_CACHE` (ON —
tries ccache, then sccache, then buildcache; override with `COMPILER_CACHE_PROGRAM`),
`MUSE_GENERATE_COMPILE_COMMANDS` (OFF). Also `MUE_DOWNLOAD_SOUNDFONT` (ON),
`MUE_RUN_LRELEASE` (ON), `MUE_RUN_WINDEPLOYQT` (OFF), `MSS_USE_SYSTEM_MNXDOM` (OFF,
for distro maintainers).

### `buildscripts/`

- `cmake/` — `SetupDependencies.cmake` (guards the submodule, wires muse_deps),
  `DependencyManifest.cmake` (requires `ogg`, `vorbis`, `flac`, `opus`, `libsndfile`;
  source-builds `liblouis`; pulls `nlohmann_json` + `json_schema_validator` for MNX),
  `PrepareDepsSources.cmake` (offline/distro prefetch, exposed as the
  `prepare_deps_sources` target), `DownloadSoundFont.cmake`, `GetUtilsFunctions.cmake`.
- `ci/` — `linux/`, `macos/`, `windows/`, `wasm/`, `backend/`, `withoutqt/`, `vtests/`,
  `testflow/`, `translation/`, `release/`, and `tools/` (env-stamp generators that
  write `build.artifacts/env/*.env` for the workflows to read back).
- `packaging/` — `Linux+BSD/` (AppImage, `.desktop`, MIME, appdata), `macOS/`
  (`Info.plist`, entitlements, DMG background), `Windows/` (Installer, PortableApps).
- `tools/` — `cmake_wrapper.bat` (MSVC), `codestyle/` (uncrustify fetch/apply).

### CI

22 workflows. `build_all.yml` is the orchestrator: it generates one shared build number
then fans out across `backend linux_arm64 linux_x64 macos windows_arm64 windows_x64
windows_portable wasm utest vtest`. `build_nightly.yml` runs on two crons — 03:00 for
`main` and 04:00 for `CURRENT_RELEASE_BRANCH: 4.7` — and is guarded by
`github.repository == 'musescore/MuseScore'` so forks don't fire it.

Two PR checks worth knowing about:

- `check_codestyle.yml` runs the uncrustify checker over `./src/` and
  `./muse/framework/` separately.
- `check_submodules.yml` guards the `muse_framework` pin: the `.gitmodules` URL must be
  the canonical upstream ("Did you accidentally leave it pointing at a fork?"), and the
  pinned SHA must be an ancestor of upstream `main`. Relevant if you use the `wf`
  script's `use-fork` task.

Also `build_without_qt.yml`, which verifies the core libraries still compile Qt-free.

### Code formatting

`./hooks/install.sh` installs a pre-commit hook that formats staged files with
**uncrustify** via `tools/codestyle/tidy_file.sh`. It stashes unstaged changes, tidies
staged and unstaged copies in parallel, re-adds the staged ones, then restores. Honors
per-directory `.untidy` opt-out markers.

## Conventions to follow when editing

- **License header.** Every source file starts with the GPL-3.0-only block plus the
  `MuseScore-Studio-CLA-applies` line. Copy an existing one and update the year.
- **Namespaces.** `mu::engraving`, `mu::notation`, `mu::project`, `mu::iex::<format>`,
  `mu::playback`; framework code is `muse::*` (`muse::audio`, `muse::mpe`,
  `muse::modularity`).
- **Option prefixes.** `MUE_*` / `MUSESCORE_*` for app options, `MUSE_*` /
  `MUSE_MODULE_*` for framework options. Don't mix them.
- **`#pragma once`** in newer headers; older ones may still use include guards.
- **`//! NOTE:`** / **`//! TODO`** is the house comment style for explanatory notes.
- Cross-module access goes through an `i*.h` interface resolved from the IoC container.
  Never include another module's `internal/` headers.
- Document-scoped services belong on a context object (`newContext`), not in
  `globalIoc()`.
- QML view models go next to their `.qml` file in `qml/MuseScore/<Module>/`, not in a
  `view/` directory.
- New module → add the option in the root `CMakeLists.txt`, a `<name>module.cpp`, a
  stub in `src/stubs/` if it is optional, and an `addModule()` call in
  `src/app/appfactory.cpp`.
- Formatting is enforced. Install the hook (`./hooks/install.sh`) or CI will fail you.

## AI contribution policy

`AI_POLICY.md` is binding for anything submitted upstream. Summary:

- AI assistance is **allowed** for code, tests, docs, and translations; the same bar
  for correctness, licensing, and review applies regardless.
- You are accountable: read and understand all generated output before requesting
  review, and be able to answer questions on it. Write PR descriptions yourself.
- Do not paste maintainer feedback into a tool and submit its reply.
- A human must review and approve any agent action that opens PRs, files issues, posts
  review comments, or pushes.
- **Disclose** substantial AI assistance: tick the PR template field and add an
  `Assisted-by: <tool> <version>` commit trailer. Use `Assisted-by:`, never
  `Co-authored-by:` — a tool cannot sign the CLA. Not needed for autocomplete,
  boilerplate, or light copy-editing. When in doubt, disclose.
- AI output does not make third-party material safe to relicense; do not use tools to
  launder licensed code.

## Where to start, by task

| Goal | Start here |
| --- | --- |
| Score data model | `src/engraving/dom/` — `score.h`, `measure.h`, `segment.h`, `note.h` |
| CC / automation curves | `src/engraving/automation/automationtypes.h`, then `internal/scoreautomationcontroller.cpp` |
| Dynamics and hairpins | `src/engraving/dom/dynamic.h`, `hairpin.h`, `dynamichairpingroup.h` |
| Layout / horizontal spacing | `src/engraving/rendering/score/horizontalspacing.cpp`, `scorehorizontalviewlayout.cpp` |
| Score canvas UI | `src/notationscene/qml/MuseScore/NotationScene/NotationView.qml` + `notationpaintview.cpp` |
| Score → audio events | `src/engraving/playback/` (`metaparsers/`, `renderers/`) |
| Playback UI, mixer | `src/playback/internal/playbackcontroller.cpp`, `qml/.../MixerPanel.qml` |
| File format support | `src/importexport/<format>/` |
| Save/load, versioned read | `src/engraving/rw/` (+ its `README.md`) |
| Undo commands | `src/engraving/editing/` and `editing/transaction/` |
| Properties panel entries | `src/propertiespanel/qml/MuseScore/PropertiesPanel/notation/<type>/` |
| Wiring a new module | `src/app/appfactory.cpp` + root `CMakeLists.txt` |
| Build config logic | `SetupConfigure.cmake` |

---

*Survey was read-only; no repository files were modified outside `agentdocs/`.
Framework details verified against the checked-out `muse` submodule at `585221a5d3`.*

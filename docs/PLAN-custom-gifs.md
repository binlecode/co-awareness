# PLAN — First-Class Custom GIF Presets (R9)

Status: implementation-ready proposal, grounded 2026-09-10. No runtime code has landed yet.

Lifecycle: when R9 is implemented and verified, distill the durable result into
`docs/ARCHITECTURE.md`, update the public surfaces, remove R9 from `docs/ROADMAP.md`, and immediately
`git rm` this file. This plan is not an as-built claim.

---

## 1. Decision summary

R9 turns the existing one-launch custom GIF path into a reusable local preset system without adding
a package, app bundle, network service, watcher, daemon, or privileged component.

The shipped user flow will be:

1. A user with an already compliant GIF may place `my-runner.gif` directly in
   `~/.config/menubar-load-runner/presets/` (or the equivalent directory below
   `$XDG_CONFIG_HOME`). It appears as `My Runner` under `Presets` on the next launch or after
   `Refresh Custom Presets`.
2. A user who wants validation and normalization chooses `Presets ▸ Import GIF…`, selects a local
   GIF, and receives an atomically installed custom preset which is immediately selected.
3. A terminal user may run `menubar-load-runner --import-preset /path/to/my-runner.gif`. The command
   performs the identical import, prints the installed identity/path and normalized dimensions, and
   exits without creating a status item.
4. The existing `menubar-load-runner /path/to/file.gif` path remains available when the user wants
   to run art without installing it as a reusable preset.

The accepted first-class format is an animated GIF with at least two frames. Static raster images,
one-frame GIFs, APNG, animated WebP, video, sprite sheets, and remote URLs are not import formats in
this iteration. A one-frame GIF remains accepted only through the legacy raw positional-path flow so
R9 does not silently break an existing launch that happens to use one; it is never registered as a
custom preset because playback speed cannot create a visible load signal from one frame.

All GIF entry points share one loader and the same resource limits. Import is a normalization and
installation surface, not the only safety gate.

---

## 2. Grounded current behavior

The plan starts from these current symbols and observed behaviors:

- `Config.presetOrPath` accepts either a built-in key or an expanded local path. A raw custom path
  is not added to `allPresets`.
- `MenuBarLoadRunnerApp.init(config:)` decodes `gifs/presets.json`, constructs the immutable
  `allPresets`, resolves a built-in key before treating the argument as a path, and gives an unknown
  custom path the default built-in speed profile.
- `loadFrames(from:)` asks ImageIO to open the URL, iterates `CGImageSourceGetCount`, decodes frames,
  reads GIF delays, computes one alpha-union crop, then commits `frames`, `frameAspects`, and
  `baseDurations`.
- The current loader does not verify the ImageIO container UTI. An existing PNG path can therefore
  decode incidentally even though every public/error surface says GIF.
- The current loader accepts one frame, has no compressed-size/frame/dimension/decoded-pixel budget,
  and silently skips a frame when `CGImageSourceCreateImageAtIndex` returns nil. It can therefore
  turn a damaged N-frame GIF into a shorter apparently successful animation.
- The crop needs every decoded frame to determine a stable union. Full-resolution assets are held
  through that pass; repository history already measured the resulting launch/switch memory spike
  and right-sized built-in art to remove it.
- `frameDuration(from:frameIndex:)` prefers `kCGImagePropertyGIFUnclampedDelayTime`, falls back to
  `kCGImagePropertyGIFDelayTime`, uses 100 ms when neither exists, and floors the result at 20 ms.
- Playback always wraps with modulo arithmetic in `advanceFrames(now:)`; the source GIF's loop count
  is not consulted.
- `currentGifAspect()` clamps only the rendered slot to `Tuning.maxIconAspect == 6.0`. It does not
  reject wider source art, so a much wider image is squeezed into the six-to-one slot and becomes
  unnecessarily small.
- `updateRenderedFrames()` pre-rasterizes every prepared frame at the status item's backing scale;
  the live game loop then performs only a `CALayer.contents` pointer swap.
- `switchToGif(to:descriptor:)` already has transactional rollback: failed decode retains the prior
  path, descriptor, frames, delays, and frame index.
- The Presets submenu is built once from `allPresets`, items use array indices as tags, and
  `refreshPresetSelectionState()` zips the immutable descriptor and item arrays.
- Restart and Start at Login serialize `activePreset?.key ?? activeGifPath` through
  `Restarter.appArguments`. No active preset is written to `state.json`, and R9 will not change that
  state-file schema or its single-writer rule.
- The built-in assets currently range from 8 to 21 frames and remain under one million decoded
  pixel-frames each. Their largest edge is 333 px after the repository's prior right-sizing pass.
- ImageIO in the supported macOS SDK exposes the required public APIs: `CGImageSourceGetType`, frame
  properties and decode, `CGImageDestinationCreateWithURL`, per-frame GIF delay properties,
  container loop count, and `CGImageDestinationFinalize`. AppKit exposes
  `NSOpenPanel.allowedContentTypes`; Uniform Type Identifiers exposes `UTType.gif` on every supported
  macOS version.
- A local probe against the built-ins and a `gifsicle -O3`-optimized copy showed ImageIO producing
  full-canvas decoded images of identical dimensions for every frame. R9 nevertheless validates
  this invariant and rejects a decoder result that violates it rather than applying a shared crop
  rectangle to incompatible frames.

No current user changes to `state.json`, Keep Awake, telemetry, launcher compilation, or rendering
timing are in R9's scope.

---

## 3. Goals and non-goals

### 3.1 Goals

- Make local user-owned animated GIFs reusable and selectable from the existing Presets submenu.
- Provide a one-action native importer with no Homebrew or third-party converter requirement.
- Keep the custom asset directory transparent, editable, and suitable for dotfile management.
- Bound compressed input, frame count, canvas dimensions, decoded pixel work, delay range, loop
  duration, and rendered aspect before retaining frame memory.
- Make type detection content-based rather than extension-based.
- Keep a preset switch transactional: a bad new asset cannot destroy the current running animation.
- Keep import transactional: interruption/failure cannot leave a discoverable partial preset.
- Preserve the launcher's singleton-before-compile invariant and atomic binary compilation.
- Give `tests/qa.sh` a headless real-binary import path while keeping menu-only behaviors honestly
  identified as click/eyes-only.
- Keep all runtime/business logic in `MenuBarLoadRunner.swift` and all builds warning-free under
  `swiftc -O -strict-concurrency=complete`.

### 3.2 Non-goals

- No online gallery, downloader, URL import, clipboard import, or network access.
- No APNG, animated WebP, HEICS, video, sprite-sheet slicing, or arbitrary ImageIO format support.
- No generated motion from a static image. A pulse/bob/run synthesis policy would be a different
  product feature, not a file conversion detail.
- No third-party encoder, `ffmpeg`, ImageMagick, `gifsicle`, Swift package, Xcode project, or app
  bundle dependency.
- No file-system watcher. Discovery occurs at launch, after an import, and on explicit refresh.
- No recursive directory scan or nested preset packs.
- No custom speed-profile editor or sidecar metadata in the first cut. Every user preset inherits
  the manifest default's `SpeedProfile`, with `customSpeedProfile` remaining the last-resort fallback.
- No in-app delete/overwrite/rename operation. Those are destructive file-management actions and
  are left to Finder/the user's dotfile workflow.
- No active-preset field in `state.json`; no second writer to that file.
- No change to built-in `gifs/presets.json` schema or built-in default semantics.
- No attempt to preserve GIF comments, application extensions, source loop count, color tables, or
  other non-rendering metadata during import.
- No change to the runtime's high-quality interpolation, dynamic width, display link, occlusion
  pause, Reduce Motion behavior, or load-to-speed calculation.

---

## 4. User-facing acceptance contract

### 4.1 Accepted inputs

The content is accepted as a first-class custom preset only when every row below passes.

| Property | Accepted contract | Failure behavior |
|---|---|---|
| Location | A readable local filesystem file. No URL schemes or directories. | Reject. |
| Container | ImageIO reports `UTType.gif`; extension and MIME guesses are not authority. GIF87a and GIF89a are both acceptable if the decoded animation passes all other checks. | Reject with the detected type when available. |
| Extension in custom directory | Case-insensitive `.gif`. This is the discovery filter only; content is still sniffed. | Ignore and log one guidance line. |
| Frame count | 2 through 120 inclusive for import/discovery; 1 through 120 for a raw positional path. | Reject. A one-frame raw path runs as a static indicator with an explicit warning that load cannot change its appearance. |
| Compressed file size | At most 20 MiB (`20 * 1024 * 1024` bytes). | Reject before ImageIO decode. |
| Per-frame dimensions | Width and height are each 1 through 2048 px. All decoded frames must have the same full-canvas dimensions. | Reject before/at the first mismatch. |
| Total source pixel-frames | Sum of `width * height` across declared frames is at most 8,388,608 pixels, using overflow-checked integer arithmetic. | Reject before retaining decoded frames. |
| Decodability | Every declared frame must decode. Skipping a damaged frame is forbidden. | Reject with the zero-based frame index. |
| Visible content | At least one pixel in at least one frame is opaque or has alpha greater than 3. Opaque frames count as fully visible. Fully transparent frames may exist inside an otherwise visible animation. | Reject an all-transparent animation. |
| Crop | One union of all visible per-frame alpha bounds. Opaque/no-alpha content contributes its full canvas. The identical crop is applied to every frame. | Internal crop failure rejects; it never falls back to per-frame crops. |
| Cropped aspect | Width/height must be finite, positive, and no greater than 6.0. Tall/narrow art remains valid because the existing 18 pt minimum slot protects clickability. | Reject with current aspect and `6:1` guidance. |
| Transparency | Optional but recommended. Opaque backgrounds are preserved, never guessed/removed. | Accept. |
| Color | Any color model ImageIO can decode to the canonical sRGB RGBA buffer. Import may quantize to GIF's palette limits. | Reject only if canonical conversion fails. |
| Frame delay | Prefer unclamped GIF delay, then clamped delay, then 100 ms. Non-finite/non-positive values use 100 ms. Clamp each delay into 20 ms through 10 s inclusive, quantize to GIF centiseconds, then clamp again. | Accept with normalization; include the normalization count in diagnostic output. |
| Base loop duration | Sum of normalized delays must be at most 60 s. | Reject. |
| Loop count | Any source value. Runtime always loops; imported output is written with loop count `0` (continuous). | Accept and normalize. |
| Orientation | GIF frames are treated in decoded pixel orientation; no EXIF rotation surface is added. | Accept decoded orientation. |
| Interlacing/palette/disposal | Accepted when ImageIO returns complete, same-canvas composited frames. | Reject if decode/canvas invariants fail. |

### 4.2 Normalized installed output

Import never edits the source. It writes a new GIF satisfying all of these conditions:

- the alpha-union crop is baked into every frame;
- every output frame has the same dimensions;
- no output edge exceeds 384 px;
- smaller art is never upscaled;
- the cropped aspect remains unchanged;
- frames are converted through an explicit 8-bit sRGB RGBA canvas before GIF encoding;
- per-frame delays are the normalized values from the acceptance pass;
- loop count is `0` (continuous);
- frame count and order are unchanged;
- output is re-opened and fully validated before it receives its final visible filename;
- all unrelated source metadata is dropped.

The 384 px ceiling is deliberately above the current built-in maximum edge (333 px) and far above
the actual menu-bar raster for ordinary aspect ratios, while preventing a reusable preset from
retaining source-resolution art that cannot produce additional visible detail.

### 4.3 Guidance, not validation

Documentation should recommend:

- transparent background;
- a compact animation cycle rather than a long scene;
- cropped art at or below 384 px on the longest edge when hand-installing;
- base delays around 40–120 ms so changing speed remains visually meaningful;
- horizontal aspect at or below 6:1;
- artwork the user owns or has permission to use.

These recommendations do not replace the exact acceptance checks above.

---

## 5. Storage, identity, ordering, and ownership

### 5.1 Directory resolution

Add `UserPresetStore.directoryURL` with one public, deterministic rule:

1. If `XDG_CONFIG_HOME` is non-empty and absolute, use
   `$XDG_CONFIG_HOME/menubar-load-runner/presets/`.
2. Otherwise use `~/.config/menubar-load-runner/presets/`.
3. A relative `XDG_CONFIG_HOME` is invalid; warn and use the default rather than resolving it against
   an unpredictable current working directory.

Respecting the standard XDG variable makes the directory dotfile-friendly and lets `tests/qa.sh`
exercise real discovery/import side effects under `tmp/` without introducing a test-only path hook or
repurposing `HOME`.

When `scripts/install-login-item.sh` runs with a valid absolute `XDG_CONFIG_HOME`, it must include the
exact XML-escaped value in the LaunchAgent's `EnvironmentVariables`. Otherwise a custom key selected
under a shell-configured XDG root would disappear at the next login, when launchd does not inherit the
interactive shell environment. Invalid/relative values are not baked in.

The app creates the directory only for Import, Show Folder, or CLI import. A normal launch with no
directory performs no write. Directory creation uses ordinary user permissions and never requests
root.

### 5.2 Installed file layout

The first cut has exactly one file per custom preset:

```
$XDG_CONFIG_HOME/menubar-load-runner/presets/
+-- my-runner.gif
+-- pixel-dog.gif
`-- .mblr-import-<UUID>.gif    temporary; ignored by discovery
```

There is no custom manifest and no sidecar. The filename is the durable identity and the GIF is the
entire user-owned preset.

### 5.3 Public key and menu title

- A ready drop-in filename stem must match
  `[a-z0-9](?:[a-z0-9-]{0,62}[a-z0-9])?` (1–64 characters), apart from a
  case-insensitive `.gif` extension. Lowercase ASCII keys make CLI, case-insensitive filesystems, git,
  and stable sorting agree.
- Its public preset key is `custom:<stem>`, for example `custom:my-runner`.
- The `custom:` namespace makes collision with every present/future built-in key impossible without
  reserving ordinary words or changing built-in precedence.
- The menu title is the stem split on `-`, with each ASCII word capitalized: `my-runner` becomes
  `My Runner`. It does not append ` (Custom)` because the disabled Custom section label already owns
  that context. Its accessibility title is `Custom preset: My Runner`. The stored identity never
  includes display decoration.
- Import slugifies the source filename stem: lowercase; each run of characters outside ASCII
  `[a-z0-9]` becomes one `-`; leading/trailing hyphens are removed; output is truncated to 64 ASCII
  characters without leaving a trailing hyphen. If nothing remains, use `custom-gif`.
- Import never overwrites. If the slug already exists, choose `-2`, `-3`, and so on while keeping the
  complete stem within 64 characters. This also makes two simultaneous/manual name collisions
  recoverable without a destructive confirmation path.
- A manually dropped invalid filename is skipped with guidance to use Import, which will normalize
  it. The app never silently renames a file it did not create.

### 5.4 Discovery eligibility

Discovery is shallow and deterministic:

- enumerate only direct children;
- skip hidden files and `.mblr-import-*` files;
- require the filename grammar and `.gif` extension;
- require a regular file; skip symbolic links and directories;
- perform the cheap metadata preflight before creating a candidate descriptor; full decode remains a
  selection-time acceptance gate;
- sort custom descriptors by key using literal ASCII ordering;
- keep built-ins in manifest order, then custom presets, then action rows.

A custom file which fails metadata preflight never makes startup fatal; discovery logs why it was
skipped. A candidate whose later frame decode/crop fails remains transactional: the current animation
continues, the row is disabled for this registry generation, and Refresh gives a manually repaired
file a new attempt. A broken built-in manifest or active launch target retains the current fatal
behavior because those are the identity the launch explicitly requested, not optional directory
content.

### 5.5 User-data lifetime

- Git pull/self-update never touches the user directory.
- The app uninstaller does not remove custom GIFs. It should print one concise note with the retained
  directory when that directory exists.
- Login-item uninstall does not touch custom GIFs.
- No automatic garbage collection deletes user presets. Only stale app-owned `.mblr-import-*` files
  older than 24 hours are eligible for best-effort cleanup during Import/Show Folder; ordinary launch
  discovery remains read-only.

---

## 6. Presets submenu and CLI contract

### 6.1 Menu structure

Rebuild the existing `Presets` submenu into this stable order:

```
Presets
+-- <built-ins in manifest order>
+-- ------------------------------
+-- Custom                         disabled section label
+-- <custom items sorted by key>   or disabled "No Custom Presets"
+-- ------------------------------
+-- Import GIF...
+-- Refresh Custom Presets
`-- Show Custom Presets Folder
```

Menu behaviors:

- selecting a built-in/custom item uses the same transactional switch;
- clicking the already active item deliberately reloads it, so a manually replaced file can be
  picked up without restarting;
- `Import GIF…` uses `NSOpenPanel`, `allowedContentTypes = [.gif]`, one file, no directories, and a
  stable panel identifier;
- import success refreshes the registry, selects the new descriptor, resets animation timing, and
  reports no extra success modal—the immediately changed icon and selected row are confirmation;
- import failure uses the existing runtime error presentation and leaves the current animation and
  registry intact;
- Refresh rescans/rebuilds but does not automatically reload or switch the active art;
- Show Folder creates the directory if needed and opens it with `NSWorkspace.shared.open`;
- no Delete/Replace/Rename rows are added;
- all new strings live in `MenuTitle` rather than at action sites;
- action rows have accessibility titles identical to their visible titles; the disabled Custom/empty
  rows must not look selectable.

Store `presetsSubmenu` as an app property. `rebuildPresetMenu()` owns all descriptor rows and action
rows, clears/reconstructs `presetMenuItems`, and assigns tags only after `allPresets` is final for that
generation. No other function edits this submenu.

### 6.2 CLI import

Add this public command:

```bash
menubar-load-runner --import-preset /absolute/or/tilde/path.gif
```

Contract:

- exactly one source path is required;
- launch-only flags/preset positionals cannot be combined with import;
- the path expands `~` using the same Foundation rule as the current positional path;
- it imports synchronously, prints one stable summary to stdout, and exits without creating
  `NSApplication.shared` or a status item;
- success exits `0`;
- invalid syntax/unsupported input exits `2`;
- filesystem/encoder/internal failure exits `1`;
- output shape:

```
Imported custom:my-runner -> /.../presets/my-runner.gif (16 frames, 120x88 px, 0.96s)
```

Extend `Config.ParseResult` with `.importPreset(String)`. The bottom-level switch invokes the shared
importer for that case before touching AppKit application lifecycle.

The launcher detects the import flag only to force attached execution so stdout and exit status reach
the caller. It still performs its normal singleton check before `compile_if_stale`; if an instance is
running, it refuses and points the user to `Presets ▸ Import GIF…`. This preserves the mandatory
singleton-before-compile ordering and avoids a second concurrent build. `--precompile` remains the sole
mode intentionally allowed ahead of the singleton guard.

Because the launcher consumes its own flags before Swift parses the remainder, it must reject
`--import-preset` combined with `--extra` or `--precompile` itself (exit `2`) before either special
branch. `--foreground`/`--no-detach` are redundant but harmless; `--detach` is overridden so import
still runs attached. Launch flags forwarded to Swift (`--load-source`, labels, Keep Awake, a preset
positional, and so on) are rejected by the Swift import-mode exclusivity check.

Update both launcher `print_help()` and Swift `Config.printUsage()` with import syntax, the
`custom:<key>` launch form, the default directory, and the absolute `XDG_CONFIG_HOME` override. The
launcher continues to forward the actual parsing/validation to Swift; it does not duplicate
GIF/import business logic.

### 6.3 Existing direct-path behavior

- Built-in key resolution remains first.
- `custom:<key>` resolution occurs against discovered user descriptors second.
- Existing-file/raw-path handling remains third.
- An unknown ordinary bareword continues to warn and fall back to the built-in default.
- An unknown `custom:<key>` also warns and falls back to the built-in default, which makes a deleted
  custom preset in a restart/login-item command recover rather than opening a fatal nonexistent path.
- A path containing `/`, ending in `.gif`, or naming an existing file remains an explicit custom path;
  failure remains fatal at startup and transactional at runtime.
- Raw paths get the same type/resource/decode validation but permit one frame for backward
  compatibility. The warning must state that load-driven speed has no visible effect.

---

## 7. Internal architecture

### 7.1 Data flow

```
                         +----------------------+
                         | built-in manifest    |
                         +----------+-----------+
                                    |
                                    v
+----------------+       +----------------------+       +----------------------+
| user directory | ----> | PresetRegistry       | ----> | Presets submenu      |
+-------+--------+       | built-ins + custom   |       | descriptors/actions  |
        |                +----------+-----------+       +----------+-----------+
        |                           |                              |
        |                           | resolve key                  | select/import
        |                           v                              v
        |                +----------------------+       +----------------------+
        +--------------> | GIFPipeline          | <---- | NSOpenPanel / CLI    |
                         | inspect/decode/prepare|       +----------------------+
                         +-----+-----------+----+
                               |           |
                       playback|           |import
                               v           v
                    +-------------+   +--------------------------+
                    | frame arrays|   | temp GIF -> validate     |
                    | transactional|  | -> atomic rename         |
                    | commit      |   +-------------+------------+
                    +------+------+                 |
                           |                        v
                           +--------------> user directory
```

### 7.2 New/changed types

Keep all types in `MenuBarLoadRunner.swift`:

```swift
private enum PresetOrigin {
    case builtIn
    case user
}

private struct PresetDescriptor {
    let key: String
    let menuTitle: String
    let path: String
    let speedProfile: SpeedProfile
    let origin: PresetOrigin
}

private enum GIFUse {
    case builtIn
    case rawPath
    case userPreset
    case importSource
    case importVerification

    var minimumFrameCount: Int {
        switch self {
        case .rawPath: return 1
        default: return 2
        }
    }
}

private enum GIFPipelineError: Error {
    case notFound(String)
    case notRegularFile(String)
    case symbolicLinkNotAllowed(String)
    case unreadable(String)
    case fileTooLarge(actual: Int, maximum: Int)
    case cannotOpen(String)
    case wrongContainer(actual: String?)
    case invalidFrameCount(actual: Int, minimum: Int, maximum: Int)
    case missingFrameProperties(index: Int)
    case invalidDimensions(index: Int, width: Int?, height: Int?)
    case dimensionTooLarge(index: Int, width: Int, height: Int, maximum: Int)
    case canvasMismatch(index: Int, width: Int, height: Int, expectedWidth: Int, expectedHeight: Int)
    case pixelArithmeticOverflow
    case pixelBudgetExceeded(actual: Int, maximum: Int)
    case frameDecodeFailed(index: Int)
    case canonicalizationFailed(index: Int)
    case allTransparent
    case invalidAspect(Double)
    case aspectTooWide(actual: Double, maximum: Double)
    case loopTooLong(actual: TimeInterval, maximum: TimeInterval)
    case destinationCreateFailed(String)
    case destinationFinalizeFailed(String)
    case postWriteValidationFailed(String)
    case atomicInstallFailed(String)
}

private struct ImportedUserPreset {
    let key: String
    let menuTitle: String
    let path: String
    let prepared: PreparedGIF
}

private enum GIFPipeline { /* static inspect/prepare/encode helpers */ }
private enum UserPresetStore { /* directory/discover/import/key helpers */ }
```

`GIFInspection` carries the exact values needed after preflight:

```swift
private struct GIFInspection {
    let url: URL
    let fileBytes: Int
    let frameCount: Int
    let canvasWidth: Int
    let canvasHeight: Int
    let sourcePixelFrames: Int
    let durations: [TimeInterval]
    let normalizedDelayCount: Int
    let loopDuration: TimeInterval
    let declaresAlpha: Bool
}
```

`PreparedGIF` carries the exact output of full decode/preparation:

```swift
private struct PreparedGIF {
    let inspection: GIFInspection
    let frames: [CGImage]
    let durations: [TimeInterval]
    let cropWidth: Int
    let cropHeight: Int
    let preparedWidth: Int
    let preparedHeight: Int
    let aspect: CGFloat
}
```

A conversion helper produces the `[NSImage]`, repeated aspects, and delays that the existing app
arrays need. Do not store both representations longer than required.

`GIFPipelineError` has stable, specific cases and one `userMessage`, for example: not found, not a
regular file, unreadable, file too large, cannot open, wrong container, frame-count violation,
missing/invalid dimensions, dimension cap, pixel arithmetic overflow, decoded-pixel budget, frame
decode failure with index, canvas mismatch with index/actual/expected, canonical RGBA conversion
failure, all-transparent, aspect too wide, loop duration too long, destination creation/finalize
failure, post-write validation failure, and atomic rename failure.

The user-facing message stems are part of the QA contract:

| Error case | Required message stem |
|---|---|
| Missing path | `GIF file not found:` |
| Not regular/unreadable | `GIF is not a readable regular file:` |
| Disallowed symlink | `Custom presets cannot be symbolic links:` |
| Compressed bytes | `GIF is <actual> bytes; maximum is 20 MiB:` |
| ImageIO source failure | `Unable to open GIF source:` |
| Wrong container | `Not a GIF (detected <type>):` |
| Frame count | `GIF has <n> frame(s); accepted range is <min>–120:` |
| Missing/invalid properties | `GIF frame <index> has invalid dimensions:` |
| Dimension ceiling | `GIF frame <index> is <w>x<h>; maximum edge is 2048 px:` |
| Canvas mismatch | `GIF frame <index> decoded as <w>x<h>; expected <w>x<h>:` |
| Pixel budget | `GIF decodes to <n> pixel-frames; maximum is 8388608:` |
| Frame decode | `Failed to decode GIF frame <index>:` |
| Canonical conversion | `Failed to convert GIF frame <index> to sRGB RGBA:` |
| Invisible content | `GIF has no visible pixels:` |
| Aspect ceiling | `GIF's cropped aspect is <x>:1; maximum is 6:1:` |
| Loop duration | `GIF's normalized loop is <x>s; maximum is 60s:` |
| Encoder/finalize | `Failed to encode normalized GIF:` |
| Verification | `Normalized GIF failed verification:` |
| Exclusive rename | `Failed to install custom preset atomically:` |

Append safely escaped path/value detail after the stem. Tests match the stable stem and relevant
bound, not localized Foundation suffixes.

### 7.3 Constants

Add these `Tuning` values and use them from validation, import, error text, QA logs, and docs:

| Constant | Value |
|---|---:|
| `customGifMaxFileBytes` | `20 * 1024 * 1024` |
| `customGifMaxFrames` | `120` |
| `customGifMaxSourceDimension` | `2048` |
| `customGifMaxSourcePixelFrames` | `8 * 1024 * 1024` |
| `customGifMaxInstalledDimension` | `384` |
| `customGifMaxFrameDelay` | `10.0` seconds |
| `customGifMaxLoopDuration` | `60.0` seconds |
| `customPresetKeyMaxLength` | `64` ASCII characters |
| `customImportTempMaxAge` | `24 * 60 * 60` seconds |

Reuse, do not duplicate:

- `Tuning.minGifFrameDelay == 0.02`;
- `Tuning.defaultGifFrameDelay == 0.1`;
- `Tuning.maxIconAspect == 6.0`;
- `Tuning.alphaVisibleThreshold == 3`;
- `Tuning.minAspect` and rendering insets.

These are deliberately conservative menu-bar-asset limits, not general image-editor limits:

| Limit | Rationale |
|---|---|
| 20 MiB compressed | Bounds synchronous local I/O and is over 100 times the largest current built-in GIF (~192 KiB). Compressed bytes alone do not bound decode, so this is only the first gate. |
| 120 frames | More than five times the largest built-in cycle (21) while preventing thousands of per-frame objects/timing entries. At the 20 ms floor it still permits a 2.4 s 50-fps cycle. |
| 2048 px per edge | Prevents a single pathological canvas allocation before the cumulative budget can help. Menu-bar art has no use for a multi-thousand-pixel retained edge. |
| 8,388,608 pixel-frames | Bounds canonical RGBA source storage to a 32 MiB byte floor and supports the measured <96 MiB transient gate. It is eight to sixty times current built-in pixel work. |
| 384 px installed edge | Preserves at least the current built-in source resolution while bounding repeat-load memory. It remains above the normal Retina status-bar raster even for wide art. |
| 10 s per frame / 60 s loop | Avoids an accepted “animation” appearing permanently stalled while still allowing deliberate pauses. |
| 64-character key | Keeps command lines, menu identity, and filenames readable while leaving room for numeric collision suffixes. |

### 7.4 Ownership/mutability changes

- Change `allPresets` from `let` to `var` because registry refresh/import can replace it.
- Keep `defaultDescriptor` immutable and built-in-only.
- Retain built-in descriptors in `builtInPresets` so refresh never reparses or
  reorders the manifest.
- Add `presetsSubmenu` as a stored property.
- `refreshUserPresetRegistry()` is the sole writer which merges
  `builtInPresets + UserPresetStore.discover(...)` into `allPresets`.
- `rebuildPresetMenu()` is the sole writer of descriptor menu rows and `presetMenuItems`.
- Keep `invalidPresetKeys: Set<String>` as transient session state. A full-load failure adds only that
  candidate key and disables its row until Refresh; a successful load removes it. It is never
  persisted and never changes metadata preflight.
- The loaded `activePreset` may remain a descriptor no longer present on disk after Refresh; its
  already decoded frames and speed profile keep running. A later restart passes its `custom:` key,
  which safely falls back to the built-in default if still absent.
- No part of this feature calls `persistState()` or `StateStore.save()`.

---

## 8. GIF inspection and loading algorithm

One pipeline must be used by startup, switching, discovery preflight, menu import, and CLI import.
The only policy variation is the minimum frame count and whether normalized frames are encoded.

### 8.1 Phase A — filesystem preflight

Given a standardized file URL:

1. Reject a non-file URL.
2. For `.rawPath` only, preserve the user-facing/original path but resolve a symlink target before
   resource checks. For every other use, inspect the supplied path itself. Read resource values
   without loading file contents: existence, regular-file status, symbolic-link status, readability,
   and file size.
3. Reject a directory, non-regular file, unreadable file, negative/unknown size, or size over 20 MiB.
   Discovered/imported presets also reject a symbolic link. A raw positional path preserves its
   existing symlink behavior when the resolved target is a readable regular file; the loader still
   applies every content/resource check to the opened target.
4. Create `CGImageSource` with no type hint. Cache policy belongs on frame decode, not on source
   creation.
5. Require `CGImageSourceGetType(source) == UTType.gif.identifier` as CFString. Do not trust `.gif`.
6. Get the declared frame count and enforce the `GIFUse` minimum plus the common maximum.

No `CGImage` has been created at this point.

### 8.2 Phase B — metadata budget pass

For each index from `0..<count`:

1. Require `CGImageSourceCopyPropertiesAtIndex`.
2. Read `kCGImagePropertyPixelWidth` and `kCGImagePropertyPixelHeight` as positive integral values.
3. Reject dimensions over 2048 before decode.
4. Require every frame's declared dimensions to match the first frame. Record that as the expected
   full canvas.
5. Compute `width * height` and cumulative pixel-frames with overflow-reporting multiplication and
   addition. Reject above 8,388,608 immediately.
6. Read the delay. Accept `NSNumber`/`Double`; replace non-finite/non-positive values with 100 ms.
   Choose unclamped, clamped, or default in that order, clamp to 20 ms...10 s, then quantize to GIF
   centiseconds using `(delay * 100).rounded(.toNearestOrAwayFromZero) / 100`. Clamp once more after
   quantization. This produces one deterministic duration array for playback, encoding, and exact
   post-write verification.
7. Sum normalized delays and reject a total above 60 s.

This pass prevents a small compressed file with enormous logical canvases/frame counts from reaching
the allocation-heavy decode path.

### 8.3 Phase C — canonical decode

For every declared frame:

1. Call `CGImageSourceCreateImageAtIndex` with
   `[kCGImageSourceShouldCache: false, kCGImageSourceShouldCacheImmediately: false]`.
2. Failure rejects the entire GIF and names the frame index. Never `continue`.
3. Require decoded width/height to equal the inspected full canvas.
4. Draw the decoded frame into a newly allocated full-canvas `CGContext` with:
   - 8 bits/component;
   - 4 components;
   - explicit sRGB color space;
   - explicit RGBA byte order and premultiplied-last alpha;
   - transparent initial contents;
   - no float/extended-range buffer.
5. Require `context.makeImage()` and retain that canonical frame.
6. Call `CGImageSourceRemoveCacheAtIndex` after canonicalization.

Canonicalization removes the current `NSBitmapImageRep`/source-`alphaInfo` ambiguity: alpha scanning
must inspect a buffer whose layout this code created, not infer an `NSBitmapImageRep` layout from the
original image's enum.

Use an autorelease pool per frame. Given the preflight budget, synchronous execution is intentional;
no non-Sendable image objects cross actors/queues, and strict-concurrency warnings remain zero.

### 8.4 Phase D — union visibility scan

For each canonical RGBA frame:

1. If every alpha byte is `255`, contribute the full canvas immediately.
2. Otherwise scan row-by-row. A pixel contributes when alpha is greater than
   `Tuning.alphaVisibleThreshold`.
3. Merge its tight box into one union.
4. A fully transparent frame contributes nothing but remains in the animation.
5. If no frame contributes, reject as all-transparent.

The union is inclusive integer bounds. Convert to a crop rectangle once and validate it lies wholly
inside the canvas. Never crop each frame independently; that reintroduces frame wobble.

### 8.5 Phase E — aspect and output scale

From union width/height:

1. Compute `Double(width) / Double(height)` and require finite/positive.
2. Reject an aspect over `Tuning.maxIconAspect` rather than relying on the later slot clamp.
3. Compute `scale = min(1.0, 384 / max(width, height))`.
4. Compute each output dimension with deterministic rounding, minimum 1 px.
5. Use the same output size for every frame.

The total source-pixel budget bounds the expensive decode. The 384 px edge bounds retained and
installed art. No upscale means tiny pixel art remains pixel-exact until the existing status-bar
renderer performs its normal high-quality presentation scaling.

### 8.6 Phase F — crop/resize preparation

For each canonical frame, replacing the array entry so old storage is released promptly:

1. Crop with `CGImage.cropping(to:)`; failure rejects instead of falling back to the original.
2. If `scale == 1`, retain the cropped image.
3. Otherwise draw it into an equal-sized 8-bit sRGB RGBA context at the Phase E dimensions using
   high interpolation.
4. Require the resulting `CGImage`.

At completion, every prepared frame has identical dimensions. Build one repeated aspect value rather
than recomputing per-frame aspects.

### 8.7 Phase G — transactional playback commit

Replace `loadFrames(from:) -> Bool` with a result-returning preparation function plus a small commit
method:

1. Fully prepare the candidate into locals.
2. Convert prepared `CGImage`s to the `NSImage` array expected by current rendering.
3. Verify frame/image/duration counts are exactly equal and nonzero.
4. Only then assign `frames`, `frameAspects`, and `baseDurations` together.

`switchToGif` retains its rollback snapshot until commit, including `renderedFrames` and frame aspects
in addition to the arrays it already saves. The method gains `forceReload` so selecting the already
active custom item can re-read a manually changed file. After success it performs the existing
`applySizing`, first-frame render, width refresh, game-loop timing reset, and menu-state refresh.

Startup displays the typed error and exits as today. Runtime switch/import displays it and retains the
old animation.

### 8.8 Diagnostics

Add the observation-only environment flag `MENUBAR_LOAD_RUNNER_LOG_GIF=1`. On each successful full
load/import verification, emit one stable line to stderr:

```
GIF origin=user frames=16 source=120x88 crop=118x84 prepared=118x84 sourcePixels=168960 bytes=7421 duration=0.96 normalizedDelays=0 aspect=1.405 path="/.../totoro.gif"
```

On failure, the normal typed error remains sufficient; do not emit a fake success line. The flag does
not alter acceptance, frame data, timing, or selection and therefore remains non-invasive
observability under the repository's testing rules.

Use the stable origin strings `built-in`, `raw`, `user`, `import-source`, and `import-verification`.
Render `path` with Swift's escaped/debug representation so quotes, control characters, and newlines
cannot split or forge diagnostic records.

---

## 9. Import algorithm and atomicity

`UserPresetStore.importGIF(from:)` is shared by the menu and CLI.

### 9.1 Import transaction

1. Resolve/create the user preset directory.
2. Best-effort remove only `.mblr-import-*` regular files older than 24 hours.
3. Derive and uniquify the destination key without touching existing files.
4. Run the complete GIF pipeline against the source with `.importSource` (minimum two frames).
5. Create a UUID temp path inside the final directory so final movement cannot cross filesystems.
6. Create a GIF destination with `UTType.gif.identifier` and the exact frame count.
7. Set container properties to `{ GIF: { LoopCount: 0 } }`.
8. Add each prepared frame with both unclamped/clamped delay keys set to the normalized delay. Do not
   pass source metadata through.
9. Require `CGImageDestinationFinalize`.
10. End the nested preparation/encoding scope so the initial prepared frame array is released before
    opening the temp output for full verification.
11. Re-open the temp file through the complete pipeline with `.importVerification` and require:
    - GIF UTI;
    - same frame count;
    - same prepared dimensions;
    - exactly the same centisecond-normalized delays;
    - continuous-loop container property;
    - all common resource/aspect/visibility checks.
12. Confirm the chosen final destination still does not exist. If it appeared concurrently, choose
    the next numeric suffix and recompute key/path.
13. Atomically rename the temp file to `<key>.gif` within the same directory using
    `renamex_np(..., RENAME_EXCL)` and explicit filesystem representations. If the exclusive rename
    reports a collision, repeat Step 12; never use plain `rename(2)`, which could overwrite a file
    created between the existence check and rename.
14. Return `ImportedUserPreset`: key/title/path plus the `PreparedGIF` produced by post-write
    validation. Menu import refreshes the registry, resolves the installed descriptor (thereby adding
    the inherited speed profile), and commits those already verified frames instead of decoding the
    file a third time; CLI import prints the record and releases its frames on exit.

Every failure after temp creation removes that exact UUID temp file best-effort. No broad glob or
recursive deletion appears in runtime code.

### 9.2 Why one file and no metadata

A central custom manifest would make one import a two-file transaction and add a second mutable
registry schema. A sidecar would still permit half-installed pairs after a crash/manual edit. Filename
identity plus the inherited default profile gives R9 the requested reusable art with one atomic file.
Custom titles/speed profiles can be proposed later only if a concrete need justifies the extra schema
and transaction model.

### 9.3 Import/UI failure presentation

- Unsupported/static/wide/oversized inputs state the violated value and accepted bound.
- Encoding/finalization/rename failures identify the destination directory and preserve the source.
- Menu errors use `showRuntimeError`.
- CLI errors print `Import failed: <specific message>` to stderr and return the defined nonzero code.
- No failure offers to install Homebrew tooling or sends the user online.

---

## 10. Registry refresh and lifecycle semantics

### 10.1 Startup order

Inside `MenuBarLoadRunnerApp.init(config:)`:

1. Resolve repo resources and decode the built-in manifest exactly as now.
2. Build immutable `builtInPresets` with `.builtIn` origin.
3. Discover candidate user descriptors with metadata preflight only; malformed optional files
   warn/skip and full acceptance remains selection-time.
4. Merge to `allPresets`.
5. Resolve empty/default, built-in key, `custom:` key, and path in the order specified above.
6. Keep the manifest default as `defaultDescriptor` and speed fallback.

The selected active asset receives a full decode in `applicationDidFinishLaunching` as today.
Discovery does not eagerly retain frames for every custom file.

### 10.2 Refresh

`refreshUserPresetRegistry()`:

1. rescans custom files;
2. clears `invalidPresetKeys` for a fresh attempt at repaired files;
3. replaces only the user portion of `allPresets`;
4. rebuilds the submenu if it exists;
5. re-resolves the active descriptor by key/path when present, while retaining an absent active
   descriptor for the already loaded session;
6. does not auto-switch or auto-reload;
7. emits skip warnings once per refresh, not once per menu open.

The menu's ordinary `menuWillOpen` continues to update checkmarks/enabled states but does not touch
disk beyond `fileExists`; opening a menu must not repeatedly decode a custom directory.

### 10.3 Restart, update, and login item

- A selected discovered/imported descriptor causes restart to pass `custom:<key>`.
- Restart rescans the user directory before resolving that key.
- If the file disappeared, unknown custom key falls back to built-in default with a warning.
- A selected raw path continues to restart as an absolute path.
- Start at Login bakes the same current key/path only when the user enables/reinstalls the login item;
  changing presets later does not mutate an existing plist, matching current built-in behavior.
- `scripts/install-login-item.sh` also bakes a valid absolute `XDG_CONFIG_HOME`, so the custom key
  resolves from the same directory at login. The plist omits the variable when the default
  `~/.config` root is used.
- Git self-update cannot overwrite custom files because they live outside the checkout.
- No import is attempted during a restart/update.

### 10.4 Uninstall

The top-level uninstaller continues deleting only its checkout/symlink/LaunchAgent. If the resolved
custom directory exists, print:

```
Left custom presets in <path> (remove manually if no longer wanted).
```

It must not delete the directory, even with `--yes`; `--yes` authorizes removal of the recognized git
checkout, not separate user art.

---

## 11. Security, robustness, and performance model

### 11.1 Trust boundary

Every custom GIF is untrusted local input even though R9 has no network path. Validation therefore
precedes retained full decode, and import output is validated again before discovery.

The defense layers are:

- regular local file and compressed-byte cap;
- ImageIO-reported GIF container type;
- frame/dimension/pixel arithmetic bounds before decode;
- all-frame decode with no silent skipping;
- canonical known RGBA layout;
- visible-content and aspect checks;
- normalized retained resolution;
- single-directory temp/final atomic rename;
- no overwrite and no recursive scan;
- typed errors and transactional rollback.

ImageIO remains the system decoder; R9 does not parse GIF LZW/application extensions itself.

### 11.2 Memory bound

The 8,388,608 source-pixel budget corresponds to 32 MiB of raw RGBA bytes before row/context/ImageIO
overhead. In-place frame replacement keeps the dominant retained arrays near one source set rather
than holding full source plus full prepared copies for the entire pass. `kCGImageSourceShouldCache`
is false and per-index cache removal occurs after canonicalization.

The implementation must measure peak footprint with the largest accepted fixture. The land gate is:

- import/load completes without allocation failure;
- peak physical footprint attributable to the operation stays below 96 MiB above the idle baseline;
- prepared steady-state returns near the normal AppKit baseline plus status-sized rendered frames.

If measurement exceeds the gate, reduce `customGifMaxSourcePixelFrames`; do not move the decoder onto
an unsafe actor boundary or raise the budget to make a fixture pass.

### 11.3 Runtime cost

- no directory watcher or periodic scan;
- discovery performs metadata preflight, not retained full decode;
- only the active GIF is fully decoded/prepared;
- imported art is capped at 384 px before future loads;
- the existing layer-backed render loop and occlusion pause remain unchanged.

### 11.4 Filesystem races

- Discovery produces a descriptor, not a trust token; selection revalidates the file.
- Import temp names are UUID-based and hidden from discovery.
- Final destination existence is rechecked immediately before rename.
- No overwrite means an external concurrent creator wins and the importer chooses another suffix.
- A file removed after successful load does not invalidate already retained frames.

---

## 12. Implementation sequence

Implement in this order so every intermediate commit preserves current behavior and compiles cleanly.

### Step 1 — Extract a typed, bounded GIF pipeline

- Add `UniformTypeIdentifiers` import and `Tuning` limits.
- Add `GIFUse`, `GIFInspection`, `PreparedGIF`, `GIFPipelineError`, and `GIFPipeline`.
- Move duration parsing into the pipeline.
- Replace current alpha scanning with canonical RGBA scanning.
- Replace `loadFrames(from:)` mutation with prepare-then-commit.
- Apply common safety limits to built-ins/raw paths; retain the raw-path one-frame exception.
- Preserve startup/runtime error shapes where existing QA depends on `GIF file not found`.
- Add `MENUBAR_LOAD_RUNNER_LOG_GIF` summary.

Gate: strict compile, all existing `tests/qa.sh --core`, GUI lifecycle/custom-path/error tests, and all
built-ins load.

### Step 2 — Add user preset storage and discovery

- Add `PresetOrigin` and origin to descriptors.
- Add `UserPresetStore.directoryURL`, filename/key/title helpers, and shallow discovery.
- Teach `scripts/install-login-item.sh` to preserve a valid absolute `XDG_CONFIG_HOME` in the plist.
- Split immutable built-ins from mutable merged `allPresets`.
- Resolve `custom:` keys at startup.
- Add custom-menu rebuilding, refresh, show-folder, empty state, and selected-row logic.
- Make selecting an already active row force a transactional reload.

Gate: isolated `$XDG_CONFIG_HOME` tests prove ordering, invalid-file skipping, custom-key launch, raw
path behavior, reload/rollback, and no launch-time directory creation.

### Step 3 — Add encoder/import transaction

- Add ImageIO GIF destination encoding and post-write validation.
- Add slug/unique-name/temp/rename logic.
- Add NSOpenPanel import and immediate selection.
- Add stale app-owned temp cleanup only on mutating folder actions.

Gate: CLI-level shared importer tests first; then manual menu chooser test.

### Step 4 — Add CLI/launcher surface

- Extend `Config.ParseResult` and parsing exclusivity.
- Add CLI output/exit codes.
- Make launcher import mode attached while leaving Swift as the parser/worker.
- Preserve singleton-before-compile; refuse CLI import while the app is already running with menu
  guidance.
- Update both help surfaces.

Gate: headless `qa.sh --core` import succeeds/fails with exact exit/output and starts no status app.

### Step 5 — Documentation, release, and retirement

- Document user contract and examples in README, without duplicating internal algorithm details.
- Add the as-built registry/import/loading architecture and exact parameter table to
  `docs/ARCHITECTURE.md`.
- Update launcher/Swift help and cover-page feature prose where appropriate.
- Add the SemVer-minor changelog entry and bump all required version surfaces.
- Run the full verification matrix.
- Remove R9 from Roadmap, then `git rm` this plan in
  the same landing change.

---

## 13. Real-binary QA plan

All checks drive the compiled `tmp/mblr-check` or the real launcher. Test fixtures are real GIF files,
not copied Swift business types and not mocks.

### 13.1 Committed fixtures

Add compact binary inputs under `tests/fixtures/custom-gifs/`:

| Fixture | Purpose |
|---|---|
| `valid-transparent.gif` | 3 equal-canvas frames; moving alpha extents prove one union crop; varied valid delays. |
| `valid-opaque.gif` | 2 opaque frames prove transparency is optional/full-canvas crop. |
| `one-frame.gif` | First-class rejection and raw-path compatibility warning. |
| `all-transparent.gif` | Visible-content rejection. |
| `too-wide.gif` | Alpha-cropped aspect just over 6:1. |
| `too-many-frames.gif` | 121 tiny real frames; rejected before retained decode. |
| `oversized-canvas.gif` | Declared edge 2049 px with tiny compressed payload; preflight rejection. |
| `pixel-budget.gif` | Valid dimensions/frame count but cumulative declared pixels exceed 8,388,608; preflight rejection. |
| `long-duration.gif` | Normalized delays sum over 60 s. |
| `corrupt-frame.gif` | Header/count exposes multiple frames but a later frame cannot decode; proves no silent skipping. |
| `optimized-disposal.gif` | Optimized subrect/disposal source whose ImageIO decoded frames remain equal full-canvas images. |

Fixtures are prepared once with external authoring tools, checked into the repository, and consumed
without those tools at test time. Record their intended dimensions/frame counts in `qa.sh` comments,
not in a second implementation of the parser.

Create wrong-type/truncated/oversize-byte cases mechanically under `tmp/` from small fixture/text data;
they do not warrant committed binaries.

### 13.2 Core/headless checks

Under an explicit `XDG_CONFIG_HOME="$PWD/tmp/qa-r9-xdg"`:

1. strict build remains warning-free;
2. `--help` lists import syntax in launcher and compiled binary;
3. CLI import of valid transparent GIF exits 0, prints `custom:valid-transparent`, and writes exactly
   one final `.gif` with no temp residue;
4. a second import chooses `-2`, does not overwrite, and both files remain decodable;
5. imported summary proves expected frame count, union-cropped/prepared dimensions, delay
   normalization count, and duration;
6. output is at most 384 px, continuous-loop, and reloads through the real pipeline;
7. static, wrong-type, corrupt, transparent, wide, frame-cap, dimension-cap, pixel-budget,
   long-duration, and >20 MiB inputs each fail with the specific message/exit code and no final/temp
   artifact;
8. an unwritable/non-directory config target fails atomically and preserves source;
9. import with launch flags/positional args fails syntax with exit 2;
10. import mode creates no `MenuBarLoadRunner` process/status lifecycle after exit;
11. importing a source named like a built-in produces a namespaced `custom:` key and leaves the
    built-in repo asset/manifest untouched;
12. GIF log output is observation-only: enabling it changes no frame/duration/result.

Core does not pretend to cover app-owned registry discovery, because that path belongs to the real
AppKit app. It covers the shared loader/encoder/store through the headless production import command;
registry resolution remains in the GUI lifecycle tier rather than being re-ported into a standalone
test.

### 13.3 GUI checks

Add to the existing WindowServer tier:

- launch by `custom:<key>` from isolated XDG directory;
- discovery accepts valid lowercase `.gif` and uppercase `.GIF`, ignores hidden/temp/non-GIF files,
  rejects invalid stems and symlinks, and sorts keys deterministically;
- `custom:<key>` resolves while a missing custom key follows the warning/default path;
- an ordinary launch with no custom directory performs no write/creation;
- launch by raw one-frame path and observe warning plus successful lifecycle;
- launch/switch all built-ins under the new loader;
- load transparent, opaque, tall, and 6:1 boundary fixtures and assert `LOG_GIF`/`LOG_SLOTS` shape;
- missing/damaged active custom key falls back as specified;
- bad raw path retains the existing `GIF file not found` contract;
- runtime reload failure leaves prior frames/path/profile/checkmark usable;
- successful custom selection advances at least two distinct frame indices;
- normalized delays still respond to load/fixed speed and respect the 20 ms runtime floor;
- refresh after external add/remove changes menu registry without auto-switching the loaded art;
- restart carries `custom:<key>` and returns with exactly one instance;
- Start at Login serializes the custom key, with the existing destructive launcher tier remaining
  opt-in.

### 13.4 Menu/click and eyes-only checks

Menu structure can be diffed through the existing Accessibility dump because Presets is one submenu
deep. Record click-only residue honestly:

- NSOpenPanel filters for GIF and single selection;
- Cancel changes nothing;
- successful Import immediately changes art and selection without a redundant success alert;
- failed Import shows one useful alert and retains current art;
- Show Folder opens/creates the expected XDG/default directory;
- replacing active art manually then selecting it reloads visibly;
- transparent/color/opaque art remains legible on both light and dark menu bars;
- very tall and exactly 6:1 art look correctly fitted;
- imported palette quantization is acceptable on representative color/alpha fixtures.

Anything Accessibility cannot prove remains a manual release check after R9 lands; no
fake PASS is added.

### 13.5 Performance checks

Using the real binary and largest accepted fixture:

- measure idle baseline, peak during load/import, and settled footprint;
- assert/record the <96 MiB above-baseline peak gate;
- confirm ordinary imported preset steady-state remains in the existing expected band;
- confirm visible animation CPU remains comparable to a built-in of the same frame count/delay;
- confirm occlusion still stops the driver completely;
- confirm startup with many valid custom descriptor files performs only metadata preflight and does
  not retain every animation.

### 13.6 Launcher invariants

Extend `tests/qa.sh --launcher` to prove:

- ordinary duplicate launch still checks singleton before compile;
- CLI import with a running instance refuses before compile and points to menu import;
- `--import-preset` combined with `--extra` or `--precompile` exits 2 before compile; detach is forced
  back to attached import execution;
- CLI import with no instance compiles atomically if stale, runs attached, exits, and leaves no app;
- `--precompile` behavior/order is unchanged;
- concurrent ordinary launch/import requests never write the same Mach-O output;
- an import never edits the running binary or repo manifest;
- login-item installation XML-escapes and preserves an absolute `XDG_CONFIG_HOME`, omits it for the
  default root, and refuses to bake a relative value.

---

## 14. Documentation and compatibility changes

### 14.1 Public documentation

README should state only the user contract:

- accepted format and limits;
- recommended transparency/timing/aspect;
- drop-in directory and XDG override;
- menu/CLI import examples;
- `custom:<key>` launch example;
- raw path escape hatch/one-frame limitation;
- refresh/removal behavior;
- uninstall preserves custom art;
- local-only/no gallery and user asset-rights responsibility.

Do not duplicate the internal decode algorithm there; link to the as-built Architecture section after
landing.

### 14.2 SemVer

R9 adds a public CLI flag, environment/path convention, preset-key namespace, and observable menu
behavior. It requires a MINOR version release under the repository's public-API definition. Move all
version surfaces together and complete the architecture release hygiene sequence (`docs/ARCHITECTURE.md` § 13).

### 14.3 Backward compatibility

- all built-in keys and manifest fields remain unchanged;
- default preset remains manifest-owned;
- existing raw GIF paths/env continue to work;
- one-frame raw GIFs remain launchable with a warning;
- raw non-GIF ImageIO formats that happened to decode are intentionally rejected: they were never a
  documented contract, and retaining them would undermine the explicit GIF acceptance/safety model;
- existing restart/login absolute paths remain valid;
- `state.json` shape/version remains unchanged;
- user presets survive app source updates and uninstall.

---

## 15. Land criteria

R9 is complete only when every item is true:

- [ ] All GIF entry points use the shared typed pipeline.
- [ ] Content type, compressed bytes, frames, dimensions, total pixels, all-frame decode, visibility,
      aspect, delays, and loop duration are enforced exactly as specified.
- [ ] The current silent damaged-frame skip is gone.
- [ ] Alpha scanning uses a canonical known RGBA layout and one union crop.
- [ ] Prepared/install output never exceeds 384 px and never upscales.
- [ ] Import writes and post-validates a temp GIF, then atomically renames one file without overwrite.
- [ ] User directory resolution respects absolute `XDG_CONFIG_HOME` and defaults to `~/.config`.
- [ ] Custom keys use `custom:` and cannot collide with built-ins.
- [ ] Presets submenu shows deterministic built-in/custom/action groups and supports explicit refresh.
- [ ] Menu and CLI import share implementation; CLI mode launches no app/status item.
- [ ] Launcher singleton-before-compile and atomic Mach-O replacement are unchanged and tested.
- [ ] Failed startup/switch/import behavior is specific and transactional.
- [ ] Restart/Start at Login/custom deletion semantics match this plan.
- [ ] No state schema/write path, package, build system, privilege, watcher, network path, or mock is
      introduced.
- [ ] `swiftc -O -strict-concurrency=complete` produces zero output/warnings.
- [ ] `tests/qa.sh --core`, full GUI suite, and `tests/qa.sh --launcher` pass on applicable hardware;
      environmental gaps report NOTE.
- [ ] Peak-memory and occlusion gates pass on real binary/fixtures.
- [ ] README/help/changelog/cover/version and as-built Architecture are synchronized.
- [ ] R9 is removed from Roadmap, and this plan is
      deleted in the landing change.

No design question remains open for implementation. A request for static-image animation, another
container, editable metadata, overwriting/deleting UI, recursive packs, or remote art is a new scoped
proposal rather than an implicit extension of R9.

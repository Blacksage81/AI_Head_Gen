# AI Head Generator

A self-contained Blender extension that generates a textured 3D head
mesh from photos, fitted to Google's GNM parametric head model, for
export into Character Creator 4.7's Headshot 2 plugin. No other add-ons
required - heavy dependencies (MediaPipe, GNM) run in separately-managed
Python environments the extension installs and talks to over local
HTTP, keeping Blender's own embedded Python untouched.

Licensed under GPL-3.0-or-later (see `LICENSE`) - required by Blender
Foundation policy for any add-on that touches `bpy`, independent of the
Apache-2.0 license on the GNM model itself (attribution required,
downloaded from Google's own repo at install time - never bundled here).

**Current status**: real, working landmark detection (MediaPipe) and
identity fitting (GNM, bounded to the model's documented parameter
range, with head-pose rotation solved for) against a single uploaded
photo - producing a recognizable, human-proportioned head, not a
placeholder. Expression fitting, multi-image support, the vision-LLM
refinement loop, texture projection, and OBJ/MTL export are not yet
implemented - see "Deliberately not yet implemented" below.

## Installing (Blender 4.2+, tested on 5.1)

1. Zip this folder's *contents* (not the folder itself) or point Blender's
   Extensions installer at this directory.
2. Edit > Preferences > Get Extensions > install from disk.
3. Enable the add-on; a "Head Gen" tab appears in the 3D viewport sidebar (N-panel).
4. In Preferences, use "Install GNM Environment" and "Install Landmark
   Environment" once each - these download and set up the companion
   Python environments the extension needs (see "What actually works"
   below for what each contains).

## What actually works right now

- Add-on registers/unregisters cleanly (properties → preferences →
  operators → panels order matters, see `__init__.py`).
- Preferences panel: local/cloud mode toggle, masked API key field,
  local endpoint URL, GNM environment install button, **landmark
  environment install button**.
- "Install GNM Environment" button really creates a private venv (using
  Blender's own embedded Python - GNM's official package is tested
  against 3.13, so this works directly) and attempts a real
  `pip install` of Google's GNM shape package - runs in a background
  thread via `AHG_ModalBackgroundOperator` so the UI doesn't freeze.
- "Install Landmark Environment" button downloads a standalone CPython
  3.12 build (python-build-standalone), builds a venv from *that*
  interpreter, and pip-installs MediaPipe into it - all independent of
  Blender's own embedded Python. This exists because **MediaPipe has no
  Python 3.13 wheels as of this writing**, and Blender 5.1 embeds 3.13,
  so MediaPipe can't be a bundled extension wheel the way the original
  brief assumed. Cross-platform (Windows/macOS/Linux, x86_64/ARM64).
- **Landmark detection is real, not a stub.** `landmark_server.py` runs
  as a subprocess inside the landmark venv (stdlib HTTP server, never
  imported by Blender's own process directly - it imports mediapipe,
  which only exists in that venv). `landmark_client.py` starts it on
  first use, health-checks it, and talks to it over
  `http://127.0.0.1:8765`. The Face Landmarker model file
  (`face_landmarker.task`, Apache-2.0, ~separate from the pip package)
  downloads itself into a cache dir on first request. Server shuts down
  cleanly when the add-on unregisters.
- **Landmark-based identity fitting is real and produces recognizable
  results.** A 68-point reduced landmark set is corresponded to specific
  GNM template vertices via self-calibrating coarse alignment + ICP
  (confirmed against the actual GNM template's coordinate convention:
  +Z forward, -Y down, Y is the neck-to-crown axis - both empirically
  verified, not assumed). `/fit` on the GNM server solves for identity
  parameters (bounded to GNM's own documented "-3 to +3" typical range)
  plus a weak-perspective camera scale/offset **and a rigid head-pose
  rotation**, via `scipy.optimize.least_squares`. The rotation term was
  added after discovering bounded fits plateaued at a high error no
  amount of regularization tuning could fix - the signature of an
  un-modeled degree of freedom (head tilt), not a shape problem.
  Confirmed via live testing to produce an actually recognizable,
  human-proportioned head (not a neutral template, not a distorted
  mess) - see project history for the full debugging trail.
- Photo list UI: add/remove file paths via the standard file browser.
- **"Build Face Correspondence (Verify)" button** - diagnostic tool,
  drops a reference head + two marker sets (final ICP result, and the
  pre-ICP coarse alignment) into the scene so the correspondence can be
  visually sanity-checked before trusting it for fitting.

## Deliberately not yet implemented (see project brief for design)

- **Expression** - held at zero throughout; only identity is fit. This
  is why fitted heads currently show a fixed neutral expression
  (heavy-lidded/closed-looking eyes) regardless of the photo's actual
  expression.
- **Multi-image fitting** - only the first uploaded photo is used right
  now; multi-view triangulation/averaging across photos isn't wired in.
- Vision-LLM HTTP client (shared OpenAI-compatible request/response code
  for both local TextGen and cloud APIs) and the semantic-layer tool
  schema (`nudge_identity`, `nudge_expression`) - natural next stage now
  that a working baseline fit exists to refine.
- Texture projection/bake
- OBJ/MTL export

## Known open items carried over from the brief

- ~~Confirm whether `gnm/shape/pyproject.toml` has a TensorFlow-free
  install extra~~ **Resolved**: installing GNM's package normally pulled
  in both TensorFlow *and* jupyterlab (likely via its Colab/demo-viewer
  code, not anything the fitting code needs) - jupyterlab's deeply
  nested asset folders were also directly responsible for a Windows
  path-length failure that broke a reinstall's cleanup. Fixed by
  installing an explicit minimal dependency list, then GNM itself with
  `--no-deps`. The dependency list is based on a third-party
  integration's documented requirements, not verified line-by-line
  against `gnm_numpy.py`'s actual imports - if something's missing it
  will surface as an ImportError at first use.
- `AHG_OT_test_connection`'s cloud-provider URL/auth-header logic is a
  placeholder - needs real per-provider endpoints (Anthropic vs OpenAI vs
  Google differ in auth header format). **Confirmed working against a
  live local TextGen endpoint.**
- **Fit runtime vs. thoroughness tradeoff.** `max_nfev=500` was chosen
  because a live test at `max_nfev=800` (with the added rotation
  parameters) took over 4 minutes to converge - too slow for a
  responsive UI button, even though it did eventually produce a good
  result. 500 converges faster but hasn't been stress-tested across a
  range of photos; may need real progress feedback during the fit
  (not just a static "fitting..." message) if this remains slow, or a
  faster convergence strategy (analytical Jacobian, warm-starting)
  if quality suffers at the lower budget.
- Fit quality (`rmse_px` returned alongside the fitted identity) is a
  rough signal, not validated as a reliable proxy for "does this
  actually look like the photo" - two of the debugging session's fits
  had similar RMSE (~100px) despite one being clearly better than the
  other structurally (bounded-with-rotation vs bounded-without).
- Pose rotation is currently loosely bounded to ~45 degrees per axis -
  untested whether this is the right range for typical photo variation.
- **Orphaned server processes.** Neither companion server subprocess is
  tied to Blender's own lifetime at the OS level - if Blender is
  force-quit, crashes, or is closed without disabling the add-on first,
  the server keeps running and can hold files locked, blocking a later
  reinstall (observed directly: a stale landmark server held `cv2.pyd`
  locked with "Access is denied" after a Blender restart that didn't go
  through a clean unregister()). Current fix is reactive - both
  installers now kill any process found running from inside their
  target directory before creating the venv. The more thorough fix
  (tying the subprocess's lifetime to Blender's via a Windows Job
  Object, so it's killed automatically no matter how Blender exits)
  hasn't been implemented - it needs `ctypes`/`ITaskbarList`-style APIs
  or the `pywin32` package, neither of which was worth adding for this
  phase.

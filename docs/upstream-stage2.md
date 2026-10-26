# Stage 2: medium-risk upstream backports

Based on the fork's merged Stage 1 (`db8a8305`). These are selected backports from
`storytold/lightcraft` through v0.14.0, adapted to the Android engine rather than a wholesale engine
upgrade. The app version is unchanged. Render-cache version 18 invalidates old previews.

| Area | Change in everyday terms | Upstream source |
| --- | --- | --- |
| RAW previews | Use smaller suitable camera JPEGs and avoid reading sensor data when a header locates the preview. Android's grid uses the quick thumbnail job, with normal rendering on failure or edited photos. | #755 `e4a248c9`, #1410 `c6b2686b` |
| Large DNG thumbnails | Develop a smaller embedded LinearRaw copy when suitable, avoiding full-size development just for a thumbnail. Exports always use the original. | #2263 `195340da` |
| RAW development | Process AHD demosaicing in bands with less temporary work; apply camera colour tables in batches. Existing supported formats benefit; this does not add the newer ARW6 decoder. | `f4078135`, #927 `95007492`, colour subset of #1320 `6467d543` |
| Memory | Sample the guided filter's reduced coefficients without allocating full-size coefficient planes. This trades some filter CPU time for less peak memory; it is not a universal speedup. | #1976 `5b4c0e79` |
| Highlights/shadows | Keep fine texture when brightening shadows or recovering highlights. New edits use V2; existing saved edits remain V1 until explicitly upgraded or reset. The Android Undo menu offers an undoable upgrade. The CPU median filter is also optimized. | #533 `6f323e01`, #645 `70103a7e`, #2694 `6a9e1b50` |
| Noise reduction | Scale noise reduction with the original photo's dimensions so small previews better represent exports. | #1489 `de5ec99f` |
| HEIC/HEIF | Enable the decoder in Android and improve colour range/matrix handling, orientation and metadata. Decoder pinned to heic-rs 0.1.4. | #609 `78550e5d`, #1113 `465601b7` |
| Camera colour | Measured spectral colour fallback for 52 camera models and aliases, preserving accepted camera-JPEG fits. A near-black guard avoids amplifying dark noise during colour-table application. | #560 `eb2a0fde`, colour subset of #1320 |
| Lens corrections | Read and apply DNG WarpRectilinear2 chains; use CPU double-precision geometry when those polynomials need it, while retaining Android's GPU pixel limits. | #794 `8339028a` |

The non-finite DNG/demosaic guard (`89fa4e4b`) is a prerequisite for the optimized AHD code.
The existing RAW corpus test referenced a missing `docs/raf-corpus.sha256`; the upstream checksum
manifest is restored, without downloading camera files. Spectral data attribution and generator
sources are included in NOTICE, assets/ATTRIBUTION.md and tools/.

## Compatibility and boundaries

- Process versions protect the interpretation of existing editing sliders. Decoder, camera-colour,
  lens and noise-reduction correctness changes can still change the appearance of affected files;
  they are not gated behind V2. Process serialization, presets, copy/paste, reset, history and undo
  have regression coverage. See [process-versions.md](process-versions.md).
- Reduced DNG copies with unsupported row/column interleaving or their own coordinate-dependent
  corrections fall back to the original. Exports reject reduced sources already in memory.
- Header preview reads are bounded where supported; other containers still use whole-file reads.
  This does not backport the later sparse read planner. See [raw-preview-reads.md](raw-preview-reads.md).
- The Android bindings add `older_process` to DevelopState and regenerate Kotlin with the app.
- HDR editing/output, AI masks/denoise and the newer GPU zoom/cache policy are outside this stage.
  HEIC auxiliary HDR gain maps are ignored; enabling HEIC decoding does not enable HDR editing.

## Validation

Pinned Rust 1.99.0, locked dependencies:

- Full selected Rust suites: 1,168 passed, zero failed, 12 existing opt-in/diagnostic tests ignored.
  Includes 26 Android bridge tests (HEIC import/render/export and undoable process upgrade),
  502 engine tests, codec/HEIF, RAW, catalog, develop, pipeline and GPU test targets.
- Additional reduced-DNG unsupported-layout/correction regression passed after the final fallback guard.
- ARM64 Android library compilation passed.
- Clippy on all nine changed library targets with `-D warnings` passed. Test-target Clippy is not
  clean because existing and upstream synthetic tests intentionally use unwrap/expect under crate
  deny rules; production library linting is clean.
- Fourteen CPU/GPU equivalence tests passed on Mesa software Vulkan, covering V1/V2 highlights and
  shadows, noise reduction, geometry, camera tone, Dehaze, caching, masks and output spaces. Three
  large-source/banded GPU tests were excluded from that software run. Software adapters were allowed
  only for local validation; the production adapter policy is restored.

Physical-phone speed, peak memory and GPU-driver behaviour remain unmeasured. Upstream benchmarks
are not presented as Android measurements.

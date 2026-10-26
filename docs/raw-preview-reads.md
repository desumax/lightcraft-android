# RAW preview reads on Android

Unedited RAW thumbnails can use the camera's embedded JPEG. The Android bridge first runs the
engine's quick thumbnail job, falling back to the ordinary developed thumbnail if it fails. Edited
photos still render their settings; thumbnail keys include the settings hash.

For formats with a preview pointer available in the header (including RAF), `preview_window`
starts with 4 KiB and grows only far enough to cover the declared JPEG. Invalid pointers,
truncation and formats requiring deeper container traversal fall back to reading the complete file.
This backport does not include upstream's later sparse TIFF/CR3 read planner.

When several embedded JPEGs are available in the bytes read, `embedded_preview_covering` chooses
the smallest one covering the requested long edge. Candidates must match the largest preview's
aspect ratio within 1% and its number of colour components, excluding differently shaped thumbnails
and monochrome masks. If none covers the request, it uses the largest. Header reads retain the
header-selected preview; smaller-JPEG selection does not imply sparse reads of all formats.
Camera-look fitting and fallback editing sources continue to use the largest preview.

LinearRaw DNG thumbnails may develop an embedded reduced-resolution raw copy, with scaled crop
and the main image's colour metadata. Copies with incompatible shape, samples, coordinate-dependent
corrections or unsupported interleaving are skipped. Exports always reload the full original,
including small exports and exports made after a reduced source entered the thumbnail cache.

Synthetic tests cover bounded reads, malformed pointers, JPEG selection, reduced-copy selection
and export isolation. `crates/engine/tests/preview_read.rs` provides an optional real-file read
measurement through `LC_PREVIEW_RAW`; it is ignored by default. No phone timing claim is made.

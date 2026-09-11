# Tor Artifact Verification

## Required behavior

- Every downloaded Tor expert bundle must have exactly one matching entry in the tracked SHA-256 manifest.
- Packaging must fail when a required entry is absent or the downloaded bundle's digest differs.
- Bundle names in the manifest must match exactly; a digest for another platform or Tor Browser version must not satisfy verification.
- The tracked manifest must use a version-independent filename so an update replaces one file rather than creating parallel per-version checksum files.

## Manifest provenance

The manifest must contain all entries from Tor Browser's `sha256sums-signed-build.txt`, unchanged and in upstream order, preceded only by a comment identifying the exact source URL. Keeping the complete list makes review against the upstream source straightforward.

Before replacing the tracked manifest, the updater must download the corresponding detached `.asc` signature and verify it using the Tor Browser Developers signing key. The source URL and manifest entries must correspond to the configured `torbrowser.version`.

The tracked hashes remain pinned inputs to normal builds. The optional PGP verification build and the standalone Ant build provide independent verification of each selected expert bundle's detached signature.

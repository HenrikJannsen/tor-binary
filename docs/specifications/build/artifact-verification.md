# Tor Artifact Verification

## Required behavior

- Every downloaded Tor expert bundle must have exactly one matching entry in the tracked SHA-256 manifest.
- Packaging must fail when a required entry is absent or the downloaded bundle's digest differs.
- Bundle names in the manifest must match exactly; a digest for another platform or Tor Browser version must not satisfy verification.
- The tracked manifest must use a version-independent filename so an update replaces one file rather than creating parallel per-version checksum files.
- Both build entry points, the Maven build and the standalone Ant build, must use one shared definition of the rules, so a rule exists once. Which verification points each of them runs is listed below.

## Manifest format

One entry per line: the SHA-256 digest in lowercase hexadecimal, exactly two spaces, then the bundle name,
matched literally and without a path prefix. A trailing carriage return is tolerated so that a manifest
checked out on Windows still verifies. Exactly one entry may match a given bundle.

Anything else is rejected, including a tab or a single space as the separator, an uppercase digest, a digest
of the wrong length, and the `digest *name` form that `sha256sum` writes in binary mode. This is why
`tor-binary-resources/checksums/**` is excluded from end-of-line conversion in `.gitattributes`: the manifest
has to stay byte for byte as published, both for its signature and for this lookup.

## Manifest provenance

The tracked manifest is the file Tor Browser publishes as `sha256sums-signed-build.txt` for the configured
`torbrowser.version`, stored byte for byte as published, together with its detached signature
`sha256sums-signed-build.txt.asc`. Nothing may be added to or removed from the manifest, because any change
invalidates the signature and removes the only evidence that the pinned digests come from the Tor Project.

Storing the signature alongside the manifest keeps the provenance of the digests checkable at any later time.
A reviewer of an update does not have to trust that the person who prepared it verified the download; the
check can be repeated on the contents of the repository.

Because the digests are pinned in the repository, a normal build needs neither the signatures nor GPG. The
signature checks are the update-time and review-time gate, not a build-time dependency.

## Signature verification

A signature is accepted only when it was made with the pinned Tor Browser Developers key
`EF6E286DDA85EA2A4BA7DE684E2C6E8793298290`.

### The key material that may satisfy a verification

Verification must run against a keyring that holds the pinned key and nothing else, and that key must be
fetched from the configured keyserver at the start of every run of the gate.

Verifying against the caller's own keyring, with GPG allowed to retrieve missing keys while it verifies, is
not acceptable. Retrieval imports whatever key the signature names, so an attacker who publishes a key and
replaces both the artifact and its signature supplies the key that is then used to check their own
signature. Fetching the pinned key deliberately, and refusing retrieval during verification, means the only
key that can ever satisfy a verification is the pinned one.

Fetching on every run also determines how a revocation is noticed. Retrieval of missing keys never refreshes
a key that is already present, so a cached key hides a revocation published later. No verifier can discover
a revocation that its source withholds; the requirement is that the gate asks.

One fetch may serve several verifications in the same run, as long as they use the same keyring directory.
An entry point that verifies the manifest first and the bundles afterwards fetches once and reuses the
keyring; the condition for reusing it is that the same run created it.

The key must be requested by its full fingerprint. GPG then rejects a key that does not match the request,
so a keyserver cannot answer with a substitute key. It can still answer with a certificate that has a
revocation stripped out, which is the limit of what this check establishes.

### How the result is established

The exit status of `gpg --verify` must not be the basis of the decision. GPG reports success for a good
signature made by any key it holds, including a key that is expired or revoked.

A non-zero exit status from GPG stops the run before the records are examined, so the exit status can only
ever make the gate stricter. Anything in a signature file that the keyring cannot check, for example a second
signature appended to a genuine one, therefore stops the gate.

The decision must be made on GPG's status records, and those records must be matched whole: each must be
preceded by a newline or start the output. Parts of a status record contain text taken from the signing key:
the user id printed in the `GOODSIG` record is chosen by whoever created the key. A check that searches the
status output as a whole can therefore be satisfied by a key whose user id spells out the record the check is
looking for, and an unrelated key then passes as the pinned signer.

The newline must be required explicitly rather than through a start-of-line anchor. GPG escapes CR and LF in
status records, so attacker-supplied text cannot contain either, but it passes U+0085, U+2028 and U+2029
through unchanged, and Java's regular expressions count those as line terminators in multiline mode. Whether
a start-of-line anchor accepts them is an implementation detail of the regular expression engine, and the
rule must not depend on it.

The pinned fingerprint must be compared as the tenth field of the `VALIDSIG` record, and must end that
field, so that a longer fingerprint beginning with the same characters does not satisfy the comparison. The
tenth field is where GPG reports the primary key behind the signature: the primary key that certifies the signing subkey, or the
signing key itself when a primary key signs directly. The primary key is pinned rather than the signing
subkey, because the subkey is rotated while the primary key that certifies it stays the same.

Verification must require a `GOODSIG` record and must reject a `REVKEYSIG` or `EXPKEYSIG` record.

Signatures made with SHA-1 must be rejected. GPG rejects MD5 on its own but still accepts SHA-1 data
signatures; Tor signs with SHA-512, so nothing is lost by refusing it.

`KEYEXPIRED` must not be used for that decision: the Tor key emits `KEYEXPIRED` records for unrelated
retired subkeys while the current signing subkey is valid.

### Expired keys

An expired signing key stops the gate and requires a person to look.

Whether a signature was made before its key expired cannot be established from GPG's output, and a locally
held key that upstream has since extended produces the same records as a key that was genuinely retired.
An OpenPGP certificate keeps no history of its expiry either: the expiry date lives in the current
self-signature, and extending the key replaces it, so a signature that was made after the key had expired
becomes indistinguishable from one made while it was valid once the key is extended.

Refreshing the key resolves the ordinary case, which is what the gate does on its next run; anything else is
a decision about whether the key is still the right one to trust.

These two records describe the state of the key when the gate runs, not a property of the signature. The
same unchanged file moves from `GOODSIG` to `EXPKEYSIG` when the key expires, and back when the key is
extended and the keyring refreshed.

## Verification points

- The tracked manifest must be verified against its tracked signature.
- The tracked manifest must be compared with the file published for the configured `torbrowser.version` and
  must be byte for byte identical to it. This binds the tracked file to the configured release, which the
  signature alone does not establish: a genuine manifest of another release also carries a valid signature.
- Each selected expert bundle must be verified against its detached signature published next to it.
- Each selected expert bundle must be verified against the digest pinned in the tracked manifest.

The manifest is checked before the bundles are downloaded, so that an unauthenticated or outdated manifest
stops the run immediately.

| Verification point | `ant -f build.xml` | `mvn -Pcheck-pgp-signatures verify` | `mvn clean install` |
| --- | --- | --- | --- |
| Manifest signature | yes | yes | no |
| Manifest against its published source | yes | yes | no |
| Bundle signatures | yes | yes | no |
| Bundle digests | yes | no | yes |

The comparison with the published source is not optional: an entry point that omits it would accept a
correctly signed manifest of a different release.

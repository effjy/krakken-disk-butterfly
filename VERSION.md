# Krakken-Disk — What Changed in v5.0.0 (Butterfly Edition)

A comparison of `krakken-disk-v5.0.0-butterfly/` against `krakken-disk-v4.6.0-butterfly/`.

v5.0.0 is a **major release**. It drops all legacy volume/file format compatibility
and moves to a single, cleaner "V5" on-disk format whose headline feature is a
**fresh random nonce stored on every sector write**. Several robustness and
plaintext-safety fixes ship alongside it.

> ⚠️ **Breaking change:** v5.0.0 is a single-format release. It can **only** read
> v5 (`KRAKKEN5`) volumes and v5 (`KRAKKEN5`) encrypted files. Volumes and files
> created by v4.6.0 (or earlier V1/V2/V3/V4 formats) are **no longer readable**.
> Decrypt/migrate your data with v4.6.0 before upgrading.

---

## 1. New per-sector random nonce (the core v5 change)

**v4.6.0 (old behaviour):**
- The per-sector keystream nonce was *derived deterministically* from the sector
  index (`sector_nonce(idx)` = the index encoded little-endian into a zero buffer).
- Nothing random was stored per sector. On-disk each sector was
  `[ciphertext][MAC tag]`.
- Because the nonce was fixed per index, **rewriting the same sector reused the
  same keystream** — a classic two-time-pad weakness for an in-place encrypted disk.

**v5.0.0 (new behaviour):**
- Every sector write generates a **fresh 24-byte random nonce** (`SECTOR_NONCE_SIZE`,
  defined in `config.h`) and stores it on disk.
- On-disk sector record is now `[ciphertext][nonce (24B)][MAC tag]`
  (`SECTOR_RECORD_SIZE`).
- Rewriting a sector now uses a brand-new nonce, so **the keystream is never
  reused**, even when the same logical sector is overwritten repeatedly.
- The stored nonce is **authenticated by the MAC** (the MAC now covers
  `LE64(idx) || nonce || ciphertext`), so an attacker cannot swap the nonce to
  redirect the keystream of an otherwise-valid ciphertext.

This is the reason for the major version bump and the reason the on-disk layout
is incompatible with v4.6.0.

---

## 2. Cryptographic format simplifications

### Single format — legacy compatibility removed
- **Volumes:** v4.6.0 did *trial decryption* — it tried V4 (Krakken-2048) first,
  then fell back to V3 (XChaCha20-Poly1305), and accepted V2/V3/V4 version numbers.
  v5.0.0 does **one** path: derive the key, decrypt the `KRAKKEN5` header, verify
  the MAC + magic, and require `version == 5`. No fallback.
- **Files (`aead.c`):** v4.6.0 sniffed V1/V2/V3 file headers with multi-step
  read/rewind logic. v5.0.0 uses one fixed header — `[nonce 12][encrypted magic 8]`
  — recovers and verifies the `KRAKKEN5` magic, and rejects anything else.
- The sector cache lost its per-volume `master_key` *fallback* decryption attempt;
  it now authenticates only under the `file_key`.
- README dropped the "Dual-Generation Compatibility" bullet (it advertised seamless
  V3/V4 trial decryption).

### New magic & version constants (`config.h`)
- `KRAKKEN4_MAGIC "KRAKKEN4"` → `KRAKKEN5_MAGIC "KRAKKEN5"`.
- Added `SECTOR_NONCE_SIZE 24`.
- `VOLUME_VERSION` 4 → 5.
- `APP_VERSION` "4.6.0" → "5.0.0".

### Cleaner keystream / MAC derivation (`sector_cache.c`)
- Per-sector key derivation is now factored into `derive_sector_key()` and uses an
  **explicit little-endian** index (`store_le64`) so volumes are **portable across
  byte orders** (the old code hashed the raw `uint64_t` bytes — endianness-dependent).
- Keystream is now `Krakken(sector_key || nonce || "K5SEC")` — a fixed string domain
  separator (`K5SEC`) replaces the old optional single-`0x00` domain-separator byte
  and the `use_domain_separator` flag (removed from the API).
- Encryption is now centralized in one shared `sector_encrypt_write()` function used
  by **both** volume creation (`volume.c`) and the runtime cache-flush path, so the
  two code paths "can never desync." `volume.c`'s old private `encrypt_sector()` /
  `sector_nonce()` were removed in favour of this shared routine.

---

## 3. Plaintext-safety and robustness fixes (`aead.c`)

- **Large-file support:** switched from `fseek`/`ftell` (`long`) to
  `fseeko`/`ftello` (`off_t`) with `_FILE_OFFSET_BITS 64`, so files > 2 GB are
  handled correctly.
- **Hardened secure-shred:** the 3-pass file shred now checks every `fwrite`/`fflush`
  and forces data to stable storage with `fsync`, so a silently-dropped write can no
  longer leave plaintext behind.
- **No accidental plaintext to disk:** on decrypt, if no RAM-backed tmpfs
  (`/dev/shm`) is available for the plaintext temp file, v5.0.0 **refuses to proceed**
  rather than silently writing plaintext to `/tmp`. Users can opt in to on-disk temp
  storage with the new `KRAKKEN_ALLOW_DISK_TMP=1` environment variable.
- **Clearer MAC/keystream separation:** the worker's single `use_parallel_mac` flag
  was split into independent `mac_domain` (keystream domain separation) and
  `compute_mac` flags, so changing MAC behaviour can never silently alter the keystream.

---

## 4. Cosmetic / UI

- `gui.c`: header-bar subtitle now reads "v5.0.0", and the logo/SVG asset lookup
  paths point at `krakken-disk-v5.0.0-gtk4/` instead of `krakken-disk-v4.6.0-gtk4/`.
- `README.md`: title bumped to v5.0.0.

---

## Files changed

| File | Change |
|------|--------|
| `config.h` | Version bump; `KRAKKEN5` magic; new `SECTOR_NONCE_SIZE` |
| `sector_cache.h` / `sector_cache.c` | Per-sector random nonce; shared `sector_encrypt_write`; LE index; removed master-key fallback & domain-separator flag |
| `volume.c` | V5 single-format header; removed V3/V4 trial decryption; uses shared sector encryptor; `SECTOR_RECORD_SIZE` |
| `aead.c` | Single V5 file format; `off_t`/large-file support; hardened shred; tmpfs-only plaintext (with `KRAKKEN_ALLOW_DISK_TMP` opt-out) |
| `gui.c` | Version string + asset paths |
| `README.md` | Title; removed dual-generation compatibility note |

Unchanged: `fuse_mount.*`, `hybrid_kem.*`, `krakken_multi.c`, `main.c`, `Makefile`,
`new_vfs_funcs.c`, `permut2048.*`, `utils.*`, `vfs.*`, `volume.h`, and image/manual assets.

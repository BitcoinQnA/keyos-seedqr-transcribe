# Handover: Seed XOR mode

Status: ready to implement, 2026-09-02. Written for an agent with no prior context on this repo.

## What you are building

A second mode in this app that splits a BIP39 seed into N parts, each of which
is itself a valid BIP39 seed, and recombines parts back into the original. This
is Coldcard's [Seed XOR](https://github.com/Coldcard/firmware/blob/master/docs/seed-xor.md),
an open standard that explicitly invites other implementations.

Why here rather than a new app: the input path (scan a SeedQR or type words),
the output path (show words, transcribe as a SeedQR), the stateless design and
the whole UI vocabulary already exist in this repo. Splitting produces N seeds
that a user then needs to write down, which is exactly what this app already
does well.

**Do not start a new project.** Add to this one.

## Read these first

| File | Why |
|---|---|
| `README.md` | What the app is and how to build it |
| `crates/seedqr-core/src/lib.rs` | The pattern to copy: pure logic, no KeyOS deps, real tests |
| `src/app.rs` | State and Slint callback wiring |
| `ui/globals.slint` | `SeedState` (Rust pushes) and `Actions` (Rust implements) |
| `ui/components.slint` | `PageBody`, `NavHeader`, `WordPill`, `WordField`, `Subtext`, `TitleText`, `Fonts` |
| `ui/pages/entry/page.slint` | Word entry with autocomplete, keyboard handling, scrollable list |

---

## 1. The specification

### The algorithm, stated simply

XOR the **entropy byte arrays**. That is the whole thing.

```
combine(parts) -> original:
    entropy = parts[0].to_entropy() XOR parts[1].to_entropy() XOR ...
    Mnemonic::from_entropy(entropy)
```

The Coldcard document describes XOR-ing 11-bit word indices and excluding the
checksum bits. That is the same operation. `to_entropy()` returns exactly the
non-checksum bits (16, 24 or 32 bytes for 12, 18 or 24 words), so XOR-ing those
byte arrays and re-deriving the mnemonic gives an identical result with none of
the bit-fiddling. `Mnemonic::from_entropy` recomputes the checksum, which is what
the spec asks for.

**This is verified, not assumed.** Both Coldcard vectors in section 2 were
reproduced exactly with this approach (bip39 2.2, `to_entropy()` XOR,
`from_entropy()`), along with order independence and subset divergence. You are
implementing a proven algorithm, so if your vectors fail, the bug is yours.

### Rules

- Word counts: **12, 18 or 24**. All parts must be the same length as each other and as the result.
- Parts: **2, 3 or 4**.
- **N of N.** Every part is required. Any subset is itself a valid, and wrong, seed.
- Order of parts is irrelevant.
- Each part carries its own normally-computed BIP39 checksum, so a mistyped part is caught.

### Splitting

Generate N-1 parts, then set the final part to `original XOR part[0] XOR ... XOR part[N-2]`.

Two generation modes exist. **Implement random only for v1.**

- **Random**: draw entropy of matching length (16/24/32 bytes) from the TRNG, then double-SHA256 it.
- **Deterministic**: double-SHA256 over the fixed string `Batshitoshi`, the master secret, and the text `0 of 4 parts` (index is 0-based, changes per part).

The deterministic mode's exact byte serialisation is not pinned down by the
document, and getting it wrong produces parts that Coldcard will not reproduce.
If you implement it, read Coldcard's own source first and add a vector generated
by a real Coldcard. Do not infer it from the prose. See M4.

### Two things to tell the user

1. The spec recommends recording the **original's checksum word** alongside the parts, so you can confirm you have reassembled the right set. It reveals 3 bits of the real seed, and reveals that a correct subset has been assembled. Offer it, explain the trade, do not do it silently.
2. Deterministic mode lets an attacker holding all N parts confirm they were split by a Coldcard-compatible tool. Random mode does not. This is a real reason to default to random.

---

## 2. Acceptance vectors

Straight from the Coldcard document. These are the acceptance criteria: if these
pass, ship it; if they do not, nothing else matters.

### 24 words, 3 parts

```
A = romance wink lottery autumn shop bring dawn tongue range crater truth ability
    miss spice fitness easy legal release recall obey exchange recycle dragon room

B = lion misery divide hurry latin fluid camp advance illegal lab pyramid unaware
    eager fringe sick camera series noodle toy crowd jeans select depth lounge

C = vault nominee cradle silk own frown throw leg cactus recall talent worry
    gadget surface shy planet purpose coffee drip few seven term squeeze educate

A XOR B XOR C =
    silent toe meat possible chair blossom wait occur this worth option bag
    nurse find fish scene bench asthma bike wage world quit primary indoor
```

### 12 words, 3 parts

```
A = romance wink lottery autumn shop bring dawn tongue range crater truth ability
B = boat unfair shell violin tree robust open ride visual forest vintage approve
C = lion misery divide hurry latin fluid camp advance illegal lab pyramid unhappy

A XOR B XOR C =
    cannon opinion leader nephew found yard metal galaxy crouch between real trade
```

### Properties worth testing beyond the vectors

- Combining in any order gives the same result (permute the parts).
- Split then combine round-trips, for every (word count, part count) pair.
- Every generated part is itself a valid BIP39 mnemonic of the same length.
- Any proper subset of parts combines to something that is **not** the original. Guards against an off-by-one that silently drops a part.
- Mixed word counts are rejected rather than producing garbage.
- Combining a part with itself gives all-zero entropy, which is the well-known `abandon abandon ... art` seed. Verified. It does not crash, but it is a live wallet that has been swept a thousand times, so if a user manages to enter the same part twice they get a real seed and no error. Detect duplicate parts and refuse.

---

## 3. Where the code goes

### `crates/seedqr-core`

Add `pub mod xor` here, not in `src/`. This crate has no KeyOS or Slint
dependency, so it builds and tests on the host, which is the only way anything
in this project gets tested. `cargo test -p seedqr-core`.

Suggested surface:

```rust
pub enum XorError { LengthMismatch, PartCount(usize), Bip39(String) }

/// XOR any number of mnemonics together. 2..=4 parts, all the same length.
pub fn combine(parts: &[Mnemonic]) -> Result<Mnemonic, XorError>;

/// Split into `count` parts using caller-supplied entropy for the first
/// count-1. Keeping randomness out of the function keeps it testable.
pub fn split(seed: &Mnemonic, count: usize, entropy: &[Vec<u8>]) -> Result<Vec<Mnemonic>, XorError>;
```

Keep `split` free of RNG calls. The app passes bytes in; tests pass fixed bytes.

### `src/`

`src/app.rs` grows the callbacks. Randomness comes from
`security::Security::get_random()` (permission group `device-secrets.general-status`,
auto-allowed), or `getrandom`, which the template already patches to the TRNG on
device. Double-SHA256 the drawn bytes per the spec.

### `ui/pages/`

New routes. Follow the existing convention exactly, it is generated and
unforgiving:

- `ui/pages/<slug>/props.slint` holds `@rust-attr(route(path = "/<slug>"))` on a struct named `<Prefix>PageProps`
- `ui/pages/<slug>/page.slint` holds a component named `<Prefix>PagePage`
- Navigation from Slint: `Navigate.<slug>-page({ })` and `Navigate.backward()`
- `ui/gen/*` is regenerated by `build.rs`. Never edit it.

Callbacks that can fail return `bool` so the page only moves on success. See
`Actions.commit-words()` in `ui/globals.slint` and its use in the entry page.

---

## 4. Suggested flow

The welcome page (`ui/pages/page.slint`) currently offers two actions. Add a
third entry point, then:

**Split**: load a seed (reuse the existing scan and entry pages) → choose 2/3/4
parts → warn that all parts are required and that any subset is a valid decoy
wallet → show each part on its own page as `WordPill`s → offer "Transcribe this
part" into the existing SeedQR flow, per part → offer to show the original's
checksum word with the trade-off explained.

**Combine**: choose how many parts → load each one in turn (scan or type,
reusing the entry page) → show the resulting seed → offer transcription.

Reuse `/entry`, `/review`, `/format`, `/overview`, `/transcribe` rather than
duplicating them. The main work is threading "which part am I entering" through
the existing entry flow.

---

## 5. Environment

Everything runs inside the SDK Nix shell. `foundation develop` is interactive
only, so scripted work uses `nix develop --command`:

```bash
nix develop ~/.foundation/sdk/foundation-sdk-1.0.0-aarch64-apple-darwin --command foundation build
```

| Task | Command |
|---|---|
| Tests (host, no Nix needed) | `cargo test -p seedqr-core` |
| Build + sign | `nix develop <sdk> --command foundation build --release` |
| Simulator | `nix develop <sdk> --command foundation sim` |
| Single-file `.app` | `nix develop <sdk> --command foundation pack --release` |
| Environment check | `nix develop <sdk> --command foundation doctor` |

Signing identity is `passport-prime-dev`, already pinned in `app-config.toml`.
Publisher fingerprint `19be3035a84826e7732fc07f56c62175ef3a0f4a86fb63a80cf73f93c4f56cfb`.

**Gotchas that will cost you an hour each:**

- Plain `cargo check --target armv7a-unknown-xous-elf` fails with "custom targets are unstable". Use `foundation build`. Only the core crate is testable with bare cargo.
- The simulator has no camera (`camera allowed=false`), so scan paths cannot be exercised there. Hardware only.
- The simulator window cannot be screenshotted by tooling. Ask the user to look, or composite offline.
- The launcher caches the app tile per app-id. A changed icon needs a **device reboot**, not a reinstall.

---

## 6. UI conventions, which are not the SDK defaults

This matters. The ui2 theme defaults are visibly wrong on this device and were
corrected once already; do not reintroduce them.

- **Text**: use the `Fonts` global in `ui/globals.slint` (xs 18, sm 20, md 22, lg 24, xl 26), mirroring the firmware scale. Do **not** use `Theme.font-size-*`, whose smallest values are 14px, below anything Prime puts on screen.
- **Seed words**: use `WordPill`. 56px tall, pill shaped, 22px Montserrat, index in a fixed 32px column, matching KeyOS `ui/ui/widgets/seed-words.slint`. Two columns of six, paginated, is the firmware layout.
- **Pill fill** is `#5a595a` dark, `#e3e2e2` light, taken from the firmware palette. `Theme.palette-card` is the same colour as the page in dark theme and renders invisible.
- **`PageBody` is only a frame.** Pages place their own `NavHeader` then `Subtext`, in that order. An earlier version drew the subtitle before its children and it rendered above the header.
- **Back is a chevron**, via `NavHeader`. Never a button labelled "Back".
- **CTAs are Title Case** without articles, matching Foundation's own strings: `Scan SeedQR`, `Start Transcribing`, `Split Seed`.
- A fixed-width child of a `VerticalLayout` sits left. Wrap it in a centring `HorizontalLayout`.
- The on-screen keyboard covers the **bottom 306px**. Anything needed while typing must sit in the top 454px. `WordField` exposes `dismiss()` and sets an accept key, because the SDK `Input` does neither.

---

## 7. Constraints you cannot design around

- An SDK app **cannot read the device master key or any other app's seeds**. `os/security` `GetSeed` is not in a third-party permission group. So this cannot split the seed in Seed Vault. The seed must be scanned or typed in, exactly as the transcribe flow already does.
- The manifest currently grants **read-only** `os/fs`. Keep it that way. It is the app's main security claim and it is checkable from the signed artifact.
- Do not add permissions speculatively. `open_qr_scanner` needs only `ShowModal`, already in the `gui-app` template, and the type system enforces it: if a permission is missing the build fails.

---

## 8. Milestones

| M | Deliverable | Done when |
|---|---|---|
| M0 | `xor` module in `seedqr-core`, `combine` only | Both acceptance vectors in section 2 pass |
| M1 | `split` with injected entropy, plus the property tests | Round-trips for every (12/18/24) x (2/3/4) pair |
| M2 | Combine UI: choose part count, load each part, show result | Works end to end in the simulator by typing words |
| M3 | Split UI: part count, warnings, per-part display, hand off to transcription | Split then combine in the app returns the original |
| M4 (optional) | Deterministic split | Matches a part set generated by a real Coldcard. Do not ship on inference. |
| M5 | Hardware run | Split a real seed, transcribe a part, scan it back, recombine |

M0 and M1 are pure logic with no device needed, and they carry the entire
correctness risk. Do not start the UI until both vectors pass.

---

## 9. Risks

1. **Silent wrong answers.** A subtly wrong XOR still produces a valid-looking mnemonic. Nothing will look broken. The vectors in section 2 are the only real defence, so write them first.
2. **The subset footgun.** Any subset of parts is a valid seed with real funds possible on it. A user who loses one part does not get an error, they get a different wallet. The UI must state N-of-N plainly and repeatedly.
3. **Deterministic-mode interop.** Covered above. Gate on a real Coldcard vector.
4. **18-word seeds.** The spec supports them and `bip39` handles them (verified: 24-byte entropy round-trips). `security::Seed` in the SDK does **not**: `Seed::from_bytes` matches only 16 or 32 bytes and panics otherwise, so a 24-byte entropy panics the app. Note the existing SeedQR flow is already 12/24 only for the same reason. Either restrict Seed XOR to 12 and 24 and say so in the UI, or keep 18-word values entirely inside `seedqr-core` and never construct a `security::Seed` from one. Decide this at M0, not later.
5. **Scope creep into SLIP39.** Different scheme, M-of-N, not interoperable. Out of scope here.

## 10. Definition of done

- Both acceptance vectors pass in `cargo test -p seedqr-core`.
- Property tests cover ordering, round-trip, subset-is-wrong, and mixed lengths.
- `foundation build --release` is clean, permissions unchanged from today's read-only set.
- Split and combine both work end to end on real hardware, including one part transcribed to a SeedQR and scanned back.
- README updated. The "Not verified" section stays honest.

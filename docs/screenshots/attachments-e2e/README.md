# Attachment E2E — screenshot manifest

Feature branch: `feat/attachments`.

## Environment

- Server: `packages/happy-server` running on `localhost:3005`, local-disk
  storage (`./data/files/`, `isLocalStorage() = true` — prod flips to
  MinIO/S3 via env vars, no code change).
- Sim: iPhone 17 Pro, iOS 26.1, UDID `94565182-2D79-4D69-AE91-CA54BF0E4DE5`.
- App: `com.slopus.happy.dev` (Happy-Improved dev variant), local
  dev-client build via `pnpm ios`, Metro on `:8081`.
- CLI: `packages/happy-cli/bin/happy.mjs` targeting the local server via
  `HAPPY_SERVER_URL=http://localhost:3005`.
- Fixtures on the Mac at `~/tmp/happy-attachment-fixtures/` — see
  [`fixtures.json`](./fixtures.json) for the full manifest.

## Fixtures pushed to the sim

- Photos album (via `xcrun simctl addmedia`): `photo.jpg`, `photo.heic`,
  `clip.mp4`.
- Files.app → "On My iPhone" via the FileProvider LocalStorage container:
  `notes.md`, `small.pdf`, `big.pdf` (25 MB), `huge.bin` (120 MB).

## Setup evidence (this commit)

Screenshots proving the local stack came up cleanly against the
attachment-feature branch:

| # | Screenshot | What it shows |
|---|---|---|
| 00 | [00-app-fresh-launch.png](./00-app-fresh-launch.png) | Fresh `pnpm ios` dev-client build launched on iPhone 17 Pro sim. Expo dev launcher showing available Metros. |
| 01 | [01-onboarding.png](./01-onboarding.png) | Post-Keychain wipe re-launch — app boots clean, no cached prod credentials, ready to pair against `localhost:3005`. |
| 02 | [02-dev-client-launcher.png](./02-dev-client-launcher.png) | Metro handoff dialog wired to `localhost:8081` (the happy-app Metro instance, not the co-resident benjamins-mobile one on `:8082`). |

## Flow screenshots (follow-up commit)

The 10-flow interactive walkthrough is captured in a follow-up commit on
this branch. Each flow lands as `NN-<slug>-<step>.png` here plus its
corresponding CLI-side proof under [`logs/`](./logs/).

| # | Flow | Status |
|---|---|---|
| 1 | Image roundtrip (JPEG) | pending |
| 2 | PDF roundtrip (small.pdf → document block) | pending |
| 3 | Video roundtrip (clip.mp4 → `@<path>` reference) | pending |
| 4 | Source file (notes.md → `@<path>` → Read tool) | pending |
| 5 | Mixed batch (image + PDF + video + `.md` — inference proof) | pending |
| 6 | Oversize rejected (huge.bin → client-side modal, no upload) | pending |
| 7 | PDF > 22 MB downgrade (big.pdf → path route) | pending |
| 8 | HEIC image (photo.heic → iOS normalize → JPEG image block) | pending |
| 9 | Rejected roundtrip (forced `decrypt_failed` → composer chip) | pending |
| 10 | Cleanup (kill CLI → startup sweep clears stale temp dir) | pending |

## Test results (this commit)

- `packages/happy-cli` vitest — **733 passed / 18 skipped** across 77 files.
- `packages/happy-app` vitest — **689 passed** across 54 files.
- `tsc --noEmit` clean in both packages.
